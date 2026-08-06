# Crossplane on AKS — Production-Grade Demo (Composition/XRD/Claim + GitOps + Workload Identity)

> This version assumes the AKS cluster and resources from your original `crossplane-aks-lab.md` have been **deleted** and rebuilds everything from scratch — including a fresh cluster, fresh Crossplane install, and fresh providers — then goes straight into the three gaps that turn "I installed Crossplane and tested drift" into **"I designed a self-service, GitOps-driven, credential-free platform API"**: Workload Identity, XRD/Composition/Claim, and GitOps wiring.

**What you'll build:**
0. **Bootstrap** — recreate the AKS cluster, Helm, and Crossplane core + providers from nothing.
1. **Workload Identity** — authenticate the Azure provider with federated, credential-free auth from the start (no Service Principal secret step at all this time).
2. **XRD + Composition** — a self-service `XResourceBundle` API that provisions a Resource Group + Storage Account together, with enforced naming/tags.
3. **Claim + RBAC** — a namespaced object app teams use, scoped so they can request infra but never touch raw cloud credentials or Managed Resources directly.
4. **GitOps wiring** — ArgoCD reconciling all of the above from a Git repo instead of manual `kubectl apply`.

---

## Prerequisites

- Windows machine with PowerShell (elevated/Admin) or any shell with `bash` — commands below use `bash` syntax; adapt if you're on native PowerShell.
- An Azure subscription with sufficient permissions (Owner/Contributor) to create resource groups, an AKS cluster, and a Managed Identity with role assignments.
- Azure CLI (`az`) installed — see Part 0.1 if not already installed.
- A Git repo you control (GitHub is fine) to hold the GitOps manifests — this lab assumes `github.com/<you>/crossplane-platform-gitops`.

---

## Part 0 — Rebuild the base environment from scratch

### 0.1 — Azure CLI (skip if already installed from your last run)

```powershell
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -Uri "https://aka.ms/installazurecliwindowsx64" -OutFile ".\AzureCLI.msi"
Start-Process msiexec.exe -Wait -ArgumentList '/I', 'AzureCLI.msi', '/quiet'
Remove-Item ".\AzureCLI.msi"
az version
```

```bash
az login --use-device-code
az account list --output table
az account set --subscription "<your-subscription-id>"
```

### 0.2 — Recreate the resource group and AKS cluster

Note the two new flags (`--enable-oidc-issuer`, `--enable-workload-identity`) — creating the cluster with Workload Identity enabled **from the start** means you skip the "update an existing cluster" step entirely and go straight to federated auth, no Service Principal secret detour needed this time.

```bash
az group create --name rg-crossplane --location centralindia

az aks create \
  --resource-group rg-crossplane \
  --name crossplaneaks \
  --node-count 1 \
  --location centralindia \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --generate-ssh-keys
```

```bash
az aks get-credentials --resource-group rg-crossplane --name crossplaneaks --overwrite-existing
kubectl get nodes
```

### 0.3 — Helm (skip if already on PATH)

```powershell
$helmVersion = "v3.19.0"
Invoke-WebRequest -Uri "https://get.helm.sh/helm-$helmVersion-windows-amd64.zip" -OutFile "$env:TEMP\helm.zip"
Expand-Archive -Path "$env:TEMP\helm.zip" -DestinationPath "$env:TEMP\helm" -Force
Copy-Item "$env:TEMP\helm\windows-amd64\helm.exe" "C:\Windows\System32\helm.exe"
helm version
```
```bash
helmVersion="v3.19.0"
curl -L "https://get.helm.sh/helm-${helmVersion}-windows-amd64.zip" -o /tmp/helm.zip
unzip /tmp/helm.zip -d /tmp
cp /tmp/windows-amd64/helm.exe ~/helm.exe
export PATH="$PATH:$HOME"
helm.exe version
```


### 0.4 — Install Crossplane core

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm upgrade --install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace

kubectl get pods -n crossplane-system
```

### 0.5 — Install the Azure providers you'll need (resource-group + storage, scoped rather than the full family package)

```bash
cat > provider-azure-resources.yaml <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-resources
spec:
  package: xpkg.upbound.io/upbound/provider-azure-resources:v1
EOF

cat > provider-azure-storage.yaml <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-storage
spec:
  package: xpkg.upbound.io/upbound/provider-azure-storage:v1
EOF

kubectl apply -f provider-azure-resources.yaml
kubectl apply -f provider-azure-storage.yaml

kubectl get providers
kubectl get pods -n crossplane-system
```

> Note the change from `provider-family-azure` (used in your first lab) to the narrower `provider-azure-resources` + `provider-azure-storage`. This is the "install only what you need" hardening your original lab already called out as a production recommendation — a good thing to mention explicitly if asked what you'd do differently this time.

Once both providers show `INSTALLED: True HEALTHY: True`, the base environment is fully rebuilt and you're ready to go straight into Workload Identity — no Service Principal secret step required this time, because the cluster was created with `--enable-oidc-issuer --enable-workload-identity` from the start.

---

## Part 1 — Workload Identity (federated auth from a clean cluster)

**Why this matters for the interview:** static Service Principal secrets are long-lived credentials sitting in a Kubernetes Secret — a real security liability (leak, rotation burden, no expiry pressure). Workload Identity gives the Crossplane provider's pod a **short-lived, federated token** issued via the AKS OIDC issuer, with **zero secrets stored in the cluster**. This time the cluster was already created with `--enable-oidc-issuer --enable-workload-identity` in Part 0.2, so there's no separate `az aks update` step — just grab the issuer URL:

### 1.1 — Get the OIDC issuer URL

```bash
export AKS_OIDC_ISSUER=$(az aks show \
  --resource-group rg-crossplane \
  --name crossplaneaks \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

echo $AKS_OIDC_ISSUER
```

### 1.2 — Create a User-Assigned Managed Identity (UAMI) scoped down (not Owner)

```bash
az identity create \
  --name crossplane-azure-identity \
  --resource-group rg-crossplane

export UAMI_CLIENT_ID=$(az identity show \
  --name crossplane-azure-identity \
  --resource-group rg-crossplane \
  --query "clientId" -o tsv)

export UAMI_PRINCIPAL_ID=$(az identity show \
  --name crossplane-azure-identity \
  --resource-group rg-crossplane \
  --query "principalId" -o tsv)
```

**Least-privilege role assignment** — scope to a single resource group the platform team owns, *not* the whole subscription, and use `Contributor` (or a custom role) instead of `Owner`:

```bash
az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Contributor" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-crossplane-managed"
```

> 🔐 **Talking point:** "I scoped the identity to Contributor on a single resource group the platform owns, rather than Owner at subscription scope like the lab version — that limits blast radius if the identity is ever misused."

### 1.3 — Federate the UAMI to the Crossplane provider's ServiceAccount

The Azure provider runs as a pod with a specific ServiceAccount in `crossplane-system`. Find it, then create a federated credential trusting that exact SA. Since Part 0 installed the narrower `provider-azure-resources` and `provider-azure-storage` packages instead of the full `provider-family-azure`, **both** providers run their own pods/ServiceAccounts — you'll need a federated credential per provider SA (or reuse one UAMI with multiple federated credentials, one per subject):

```bash
kubectl get sa -n crossplane-system | grep provider-azure
# e.g. provider-azure-resources-xxxxxxxx
# e.g. provider-azure-storage-xxxxxxxx
```

```bash
az identity federated-credential create \
  --name crossplane-azure-resources-federated \
  --identity-name crossplane-azure-identity \
  --resource-group rg-crossplane \
  --issuer $AKS_OIDC_ISSUER \
  --subject "system:serviceaccount:crossplane-system:provider-azure-resources-xxxxxxxx" \
  --audience "api://AzureADTokenExchange"

az identity federated-credential create \
  --name crossplane-azure-storage-federated \
  --identity-name crossplane-azure-identity \
  --resource-group rg-crossplane \
  --issuer $AKS_OIDC_ISSUER \
  --subject "system:serviceaccount:crossplane-system:provider-azure-storage-xxxxxxxx" \
  --audience "api://AzureADTokenExchange"
```

> ⚠️ The `--subject` must match the provider pod's ServiceAccount **exactly**. If the provider pod restarts with a different generated SA name (some provider packages regenerate suffixes on upgrade), the federated credential breaks — pin/verify this as part of any provider upgrade runbook. Worth mentioning as an "operational gotcha" if asked.

### 1.4 — Update the ProviderConfig to use Workload Identity instead of a Secret

```yaml
apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: WorkloadIdentity
    identity:
      clientID: "<UAMI_CLIENT_ID>"
```

Apply it, then confirm there is genuinely **no Secret involved anywhere** in this setup:

```bash
kubectl apply -f provider-config-workload-identity.yaml

kubectl get secrets -n crossplane-system
# no azure-secret should exist — auth is federated end-to-end
```

Provision a quick smoke-test Resource Group to prove the federated auth path actually works before moving on to the real Composition in Part 2:

```bash
cat > ResourceGroup-smoketest.yaml <<EOF
apiVersion: azure.upbound.io/v1beta1
kind: ResourceGroup
metadata:
  name: crossplane-smoketest-rg
spec:
  forProvider:
    location: centralindia
  providerConfigRef:
    name: default
EOF

kubectl apply -f ResourceGroup-smoketest.yaml
kubectl get resourcegroup.azure.upbound.io
```

Once it reports `SYNCED: True / READY: True`, you've proven Workload Identity auth end-to-end — no Service Principal, no Secret, ever, in this run.

> ✅ **Interview soundbite:** *"This time I built the cluster with the OIDC issuer and Workload Identity enabled from creation, so I never created a Service Principal secret at all — I went straight from cluster bootstrap to federated auth. I proved it worked by provisioning a Resource Group with zero Secrets present anywhere in the `crossplane-system` namespace."*

---

## Part 2 — XRD + Composition + Claim (the self-service platform API)

This is the highest-value gap. It's the difference between "Crossplane can create a resource" and **"I built a platform API app teams can consume without touching Azure."**

We'll build `XResourceBundle` — a composite API that provisions a **Resource Group + Storage Account** together, with naming and tagging enforced centrally by the platform team (not left to whoever files the request).

### 2.1 — Confirm both providers are ready

Both `provider-azure-resources` and `provider-azure-storage` were already installed in Part 0.5 — just confirm they're healthy before defining the Composition that uses them both:

```bash
kubectl get providers
```

### 2.2 — Define the XRD (the schema of your platform API)

This is what app teams will see as their "menu" — deliberately simple, hiding Azure-specific detail.

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xresourcebundles.platform.mycompany.io
spec:
  group: platform.mycompany.io
  names:
    kind: XResourceBundle
    plural: xresourcebundles
  claimNames:
    kind: ResourceBundleClaim
    plural: resourcebundleclaims
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    location:
                      type: string
                      enum: ["eastus", "centralindia", "westeurope"]
                      default: "centralindia"
                    environment:
                      type: string
                      enum: ["dev", "staging", "prod"]
                    storageTier:
                      type: string
                      enum: ["Standard_LRS", "Standard_GRS"]
                      default: "Standard_LRS"
                  required:
                    - environment
              required:
                - parameters
```

Apply it:

```bash
kubectl apply -f xrd-resourcebundle.yaml
kubectl get xrd
```

### 2.3 — Define the Composition (how the platform team fulfils that API)

This is where naming standards, tagging, and guardrails get enforced **once**, centrally — every Claim gets them automatically instead of relying on developers to remember.

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: resourcebundle-azure
  labels:
    provider: azure
spec:
  compositeTypeRef:
    apiVersion: platform.mycompany.io/v1alpha1
    kind: XResourceBundle
  resources:
    - name: resource-group
      base:
        apiVersion: azure.upbound.io/v1beta1
        kind: ResourceGroup
        spec:
          forProvider:
            location: centralindia
          providerConfigRef:
            name: default
      patches:
        - fromFieldPath: "spec.parameters.location"
          toFieldPath: "spec.forProvider.location"
        - type: FromCompositeFieldPath
          fromFieldPath: "metadata.name"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "rg-%s"
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.parameters.environment"
          toFieldPath: "spec.forProvider.tags.environment"
        - type: FromCompositeFieldPath
          fromFieldPath: "metadata.name"
          toFieldPath: "spec.forProvider.tags.managedBy"
          transforms:
            - type: string
              string:
                fmt: "crossplane"

    - name: storage-account
      base:
        apiVersion: azure.upbound.io/v1beta1
        kind: Account
        spec:
          forProvider:
            location: centralindia
            accountTier: Standard
            accountReplicationType: LRS
          providerConfigRef:
            name: default
      patches:
        - fromFieldPath: "spec.parameters.location"
          toFieldPath: "spec.forProvider.location"
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.parameters.storageTier"
          toFieldPath: "spec.forProvider.accountReplicationType"
        - type: FromCompositeFieldPath
          fromFieldPath: "metadata.name"
          toFieldPath: "spec.forProvider.resourceGroupNameSelector.matchControllerRef"
          transforms:
            - type: string
              string:
                fmt: "true"
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.parameters.environment"
          toFieldPath: "spec.forProvider.tags.environment"
```

> **Note on realism:** exact field names/selectors vary by provider version (`provider-azure-storage` uses `resourceGroupNameRef`/`resourceGroupNameSelector` to link the Storage Account to the Resource Group created above — Crossplane resolves that dependency automatically). Double-check field names against the CRD installed in your cluster with `kubectl explain account.azure.upbound.io.spec.forProvider` before your demo — this is a good thing to actually run once so you can speak to it from real output, not the doc.

Apply it:

```bash
kubectl apply -f composition-resourcebundle-azure.yaml
kubectl get compositions
```

### 2.4 — The Claim (what app teams actually submit)

Notice how little an app team needs to know — no Azure fields, no provider config, no tagging logic. This is the self-service payoff.

```yaml
apiVersion: platform.mycompany.io/v1alpha1
kind: ResourceBundleClaim
metadata:
  name: team-checkout-storage
  namespace: team-checkout
spec:
  parameters:
    environment: dev
    location: centralindia
    storageTier: Standard_LRS
```

```bash
kubectl create namespace team-checkout
kubectl apply -f claim-team-checkout.yaml

kubectl get resourcebundleclaims -n team-checkout
kubectl get xresourcebundles
kubectl get resourcegroup.azure.upbound.io,account.azure.upbound.io
```

Once `SYNCED`/`READY` are `True` on all objects, the team has a real Resource Group + Storage Account in Azure — provisioned without ever touching the Azure Portal, CLI, or a cloud credential.

### 2.5 — RBAC: Claims yes, raw Managed Resources no

This is the governance story interviewers actually want to hear — self-service *with* guardrails, not a free-for-all.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: resourcebundle-requester
  namespace: team-checkout
rules:
  - apiGroups: ["platform.mycompany.io"]
    resources: ["resourcebundleclaims"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-checkout-can-request-infra
  namespace: team-checkout
subjects:
  - kind: Group
    name: "team-checkout-devs"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: resourcebundle-requester
  apiGroup: rbac.authorization.k8s.io
```

Note what's **absent**: no `ClusterRole` grant on `resourcegroups.azure.upbound.io` or `accounts.azure.upbound.io` for this group. App teams can only touch the namespaced Claim; raw Managed Resources and ProviderConfigs stay cluster-admin/platform-team-only.

> ✅ **Interview soundbite:** *"I scoped RBAC so application teams can only create the Claim — the abstraction — never the underlying Managed Resources or ProviderConfig. That's what makes it actually self-service with guardrails, not just 'give devs kubectl access to raw cloud CRDs.'"*

---

## Part 3 — GitOps wiring (ArgoCD reconciling Crossplane, not `kubectl apply`)

Right now everything above was applied manually. Production platforms don't do that — the Git repo is the source of truth, and ArgoCD (or Flux) continuously reconciles it, giving you **two nested control loops**: ArgoCD keeps the cluster in sync with Git, and Crossplane keeps Azure in sync with the cluster.

### 3.1 — Repo layout (a common, defensible structure)

```
crossplane-platform-gitops/
├── platform/                     # platform-team-owned, rarely changes
│   ├── crossplane-helm/
│   │   └── values.yaml
│   ├── providers/
│   │   ├── provider-family-azure.yaml
│   │   └── provider-azure-storage.yaml
│   ├── provider-config/
│   │   └── provider-config-workload-identity.yaml
│   └── compositions/
│       ├── xrd-resourcebundle.yaml
│       └── composition-resourcebundle-azure.yaml
└── claims/                       # app-team-owned, changes often
    ├── team-checkout/
    │   └── storage-claim.yaml
    └── team-payments/
        └── storage-claim.yaml
```

This split matters for the interview: **platform/** is the golden-path definition (high-blast-radius, tightly reviewed), **claims/** is the low-risk, high-frequency self-service layer (app teams can PR their own folder without touching platform internals). This is exactly the "app-of-apps" / separation-of-concerns pattern GitOps reviewers look for.

### 3.2 — Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd
```

### 3.3 — Two ArgoCD Applications (platform vs claims — different sync policies)

**Platform layer** — auto-sync but keep prune manual-review-worthy in real prod (shown here with prune on for the demo):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<you>/crossplane-platform-gitops.git
    targetRevision: main
    path: platform
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Claims layer** — same automation, but this is the folder app teams PR into:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-claims
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<you>/crossplane-platform-gitops.git
    targetRevision: main
    path: claims
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f argocd-app-platform.yaml
kubectl apply -f argocd-app-claims.yaml

kubectl get applications -n argocd
```

### 3.4 — Prove GitOps drift-correction end to end (the demo moment)

This is the sequel to the reconciliation test in your original lab — now proving it at the **Git** layer, not just the cloud layer:

1. Manually `kubectl delete resourcebundleclaim team-checkout-storage -n team-checkout` (simulating someone bypassing Git).
2. Watch ArgoCD's `selfHeal: true` notice the live state no longer matches Git and **re-create the Claim** within its sync interval.
3. Watch Crossplane then re-reconcile the Claim back into a real Azure Resource Group + Storage Account.
4. You now have a **two-layer, fully self-healing chain**: Git → ArgoCD → Claim → Crossplane → Azure.

> ✅ **Interview soundbite:** *"I wired ArgoCD in front of Crossplane with `selfHeal` enabled, then deleted a Claim directly from the cluster to simulate someone bypassing Git. ArgoCD detected the drift and reapplied the Claim from source control, and Crossplane's own reconciler then recreated the underlying Azure resources — so I had drift correction enforced at both the GitOps layer and the cloud-provisioning layer."*

---

## Full "explain it end-to-end" narrative for the interview

Use this as your rehearsed 60–90 second walkthrough if asked "tell me about your Crossplane experience":

> "I set up Crossplane on AKS in two passes. In the first pass, I proved the core value prop — installed the Azure provider with a Service Principal, provisioned a Resource Group as a raw Managed Resource, then manually deleted it in the Portal and watched Crossplane's reconciliation loop recreate it automatically, confirmed via the Azure Activity Log.
>
> For the second pass, I rebuilt the whole environment to close the gaps that matter for a real platform team — this time creating the AKS cluster with the OIDC issuer and Workload Identity enabled from the start, so I never touched a Service Principal secret at all. I federated the provider's ServiceAccount to a scoped-down Managed Identity and proved zero Secrets existed anywhere in the `crossplane-system` namespace before provisioning anything.
>
> Then I built an actual self-service API: an XRD and Composition that bundles a Resource Group and Storage Account with enforced naming and tagging, exposed to app teams as a simple Claim with three fields — location, environment, storage tier. I scoped RBAC so teams can only touch the Claim, never the raw Managed Resources or ProviderConfig.
>
> Finally, I wired the whole thing through ArgoCD instead of manual applies, split the repo into a platform layer and a claims layer, and proved self-healing at both the GitOps layer and the Crossplane layer by deleting a Claim directly and watching it get restored end-to-end without me touching anything."

That's a complete, defensible, first-hand story across auth, self-service API design, governance, and GitOps — the exact shape of what the JD is asking for.

---

## Honest caveats to have ready (shows maturity, not weakness)

- Field names in real Composition patches (`resourceGroupNameSelector`, replication-type enums, etc.) vary by provider version — always verify against the installed CRD with `kubectl explain <resource>.spec.forProvider` rather than trusting memorized YAML.
- This demo uses `patch-and-transform` Compositions (v1 style). Crossplane v2's **Functions pipeline** (Go/Python/KCL) is more expressive for conditional logic — worth naming as the next thing you'd explore, especially since your installed chart version (2.3.x) supports it.
- In real production, `ArgoCD` sync for the **platform** folder would usually be manual-approval or at minimum require PR review + CI validation (`kubectl apply --dry-run`, OPA/Conftest policy checks) before merge — full auto-sync on platform-level Compositions is higher-risk than on the claims layer.
- Federated credential subject binding to a ServiceAccount name is brittle across provider upgrades if the SA name regenerates — worth mentioning as an operational risk you'd monitor for.
