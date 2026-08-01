# Crossplane on AKS — Lab Walkthrough

> A hands-on log of installing **Crossplane** on an **Azure Kubernetes Service (AKS)** cluster and using it to provision a real Azure resource (a Resource Group), with an overview of what Crossplane is and why it's used.
---

# Crossplane: Overview, Purpose, and AKS Installation Guide

## 1. What is Crossplane?

Crossplane is a **CNCF-graduated, open-source Kubernetes add-on** that turns a Kubernetes cluster into a **universal control plane** for managing cloud infrastructure. Instead of provisioning resources via cloud consoles, CLIs, or a separate IaC pipeline, you declare infrastructure (VPCs, databases, storage buckets, Kubernetes clusters, DNS records, etc.) as Kubernetes custom resources (YAML manifests). Crossplane's controllers continuously reconcile the real-world cloud resource to match that declared state — the same control-loop model Kubernetes uses for pods and deployments.

In short: **"kubectl apply" for your cloud infrastructure, not just your containers.**

Key building blocks:
- **Providers** – plugins that talk to a specific cloud/service API (AWS, Azure, GCP, GitHub, SQL, etc.) and expose their resources as Kubernetes CRDs called **Managed Resources (MRs)**.
- **Managed Resources (MRs)** – the lowest-level representation of a single cloud resource (e.g., an Azure Resource Group or a Storage Account).
- **Compositions & Composite Resource Definitions (XRDs)** – let platform teams bundle multiple MRs into a single reusable, higher-level API (e.g., a custom `AKSCluster` XR that wraps a VNet, subnet, and AKS cluster together).
- **Claims** – the self-service interface application teams use to request infrastructure without knowing the underlying cloud details.
- **Functions** (Crossplane v2) – composition logic written as small functions (e.g., in Go, Python, or KCL) instead of static patch-and-transform YAML, enabling more expressive composition logic.

## 2. Why Crossplane is Needed

| Problem in traditional IaC / cloud ops | How Crossplane addresses it |
|---|---|
| Infra pipelines (Terraform, ARM, CloudFormation) are separate from app deployment pipelines | Infra and apps are managed through the same Kubernetes API and GitOps workflow |
| Manual drift correction; state files can get out of sync | Continuous reconciliation actively corrects drift, similar to how a Deployment controller maintains pod count |
| Developers need deep cloud-provider knowledge to request infrastructure | Platform teams build self-service APIs (Compositions/Claims) so developers just ask for "a database" |
| Multi-cloud consistency is hard (different tools per cloud) | One control plane, one API pattern (Kubernetes CRDs), across AWS/Azure/GCP/on-prem |
| State files, locks, and pipeline runners to manage | No external state store — the cluster **is** the state |

## 3. Purpose / Goal

- Provide a **Kubernetes-native, declarative control plane** for infrastructure of any kind — cloud, SaaS, or on-prem.
- Enable **Platform Engineering**: internal platform teams define opinionated, secure, compliant infrastructure APIs; application teams self-serve without needing cloud IAM/console access.
- Unify **GitOps** for both application and infrastructure lifecycle (a single Git repo, a single reconciliation loop, e.g. via ArgoCD/Flux).
- Reduce **vendor lock-in** to a single IaC tool by expressing infra abstractly and swapping providers under the hood.

## 4. Specialization

Crossplane specializes in:
- **Continuous reconciliation** of infrastructure state (not one-shot apply/plan like Terraform).
- **Composable, reusable infrastructure APIs** (XRDs/Compositions) tailored per organization.
- **Multi-cloud and hybrid provisioning** from a single control plane.
- **RBAC-based self-service**, using native Kubernetes RBAC to control who can request what infrastructure.
- **Package management** for providers/functions/configurations distributed as OCI images (`xpkg`).

## 5. Competitors / Alternatives

| Tool | Model | Notes |
|---|---|---|
| **Terraform / OpenTofu** | Plan-apply, state file | Most widely adopted; huge provider ecosystem; not continuously reconciling by default |
| **Pulumi** | Imperative-style IaC using real programming languages | Good developer ergonomics, state managed by Pulumi service or self-hosted |
| **AWS CDK / Azure Bicep / ARM / Google Deployment Manager** | Cloud-native, single-vendor IaC | Best for single-cloud shops, limited multi-cloud reuse |
| **Cluster API (CAPI)** | Kubernetes-native, but scoped mainly to cluster lifecycle | Narrower scope than Crossplane (K8s clusters only) |
| **Kubernetes Config Connector (KCC)** | Google's GCP-only equivalent of Crossplane | Single-cloud (GCP) |
| **Azure Service Operator (ASO)** | Microsoft's Azure-only equivalent | Single-cloud (Azure), tightly integrated with AKS/ARM |

Crossplane differentiates itself by being **cloud-agnostic**, **CNCF-governed** (vendor neutral), and built around **composable abstractions** rather than raw 1:1 resource mapping only.

## 6. Necessity — When You Actually Need It

Crossplane makes the most sense when:
- You're building an **internal developer platform (IDP)** and want self-service infra with guardrails.
- You manage **multi-cloud or hybrid** environments and want one consistent API model.
- You already run **GitOps** (ArgoCD/Flux) and want infra reconciliation to work the same way as app deployments.
- You need **continuous drift correction**, not just periodic `terraform apply`.

It may be overkill for small teams managing a handful of static resources — Terraform/OpenTofu may be simpler there.

## 7. Future Adoption

- **CNCF Graduated (2024)** status signals long-term stability and broad governance — a strong trust signal.
- **Upbound** (commercial backer) continues investing in **Upbound Cloud**, a managed Crossplane control-plane offering, plus the **Upbound Marketplace** for providers/functions.
- **Crossplane v2** introduced first-class **Functions** for composition logic (more expressive than patch-and-transform), suggesting Crossplane is moving toward a full "infrastructure programming" model.
- Growing traction in **platform engineering** circles (Backstage + Crossplane combos are common) as organizations formalize internal developer platforms.
- Native cloud "Config Connector"-style tools (ASO, KCC) remain single-cloud competitors, but Crossplane's neutral CNCF governance and multi-cloud reach continue to make it attractive for platform teams standardizing across providers.

---

---

## 🧰 Prerequisites

- Windows machine with PowerShell (elevated/Admin access)
- An Azure subscription with sufficient permissions (Owner/Contributor)
- Internet access to reach `charts.crossplane.io`, `xpkg.upbound.io`, and Azure endpoints

---

## Step 1 — Install Azure CLI (`az`) on Windows

Reference: [Microsoft Docs – Install Azure CLI on Windows](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest&pivots=msi-powershell)

Run the following in an **elevated PowerShell window (Run as Administrator)**:

```powershell
# Suppress progress bar noise during download
$ProgressPreference = 'SilentlyContinue'

# Download the Azure CLI MSI installer
Invoke-WebRequest -Uri "https://aka.ms/installazurecliwindowsx64" -OutFile ".\AzureCLI.msi"

# Run the installer silently
Start-Process msiexec.exe -Wait -ArgumentList '/I', 'AzureCLI.msi', '/quiet'

# Clean up the installer file
Remove-Item ".\AzureCLI.msi"
```

Verify the installation:

```powershell
az version
```

Log in to Azure (opens a device-code flow — good for headless/remote sessions):

```powershell
az login --use-device-code
```

List all subscriptions you have access to:

```bash
az account list --output table
```

Show the currently active subscription:

```bash
az account show --output table
```

---

## Step 2 — Create a Resource Group for the AKS Cluster

If the resource group doesn't already exist, create it:

```bash
az group create --name rg-crossplane --location centralindia
```

---

## Step 3 — Create a Basic AKS Cluster

Spin up a single-node AKS cluster in Central India (fine for a lab; use more nodes for production):

```bash
az aks create \
  --resource-group rg-crossplane \
  --name crossplaneaks \
  --node-count 1 \
  --location centralindia \
  --generate-ssh-keys
```

---

## Step 4 — Connect `kubectl` to the AKS Cluster

Download cluster credentials and merge them into your local kubeconfig:

```bash
az aks get-credentials --resource-group rg-crossplane --name crossplaneaks
```

```text
Merged "crossplaneaks" as current context in C:\Users\abc\.kube\config
```

Sanity-check that the node is up:

```bash
kubectl get node
```

```text
NAME                                STATUS   ROLES    AGE     VERSION
aks-nodepool1-11095005-vmss000000   Ready    <none>   2m49s   v1.35.6
```

---

## Step 5 — Install Helm

Crossplane is distributed as a Helm chart, so Helm v3 is required first:

```powershell
$helmVersion = "v3.19.0"

# Download the Helm release archive for Windows
Invoke-WebRequest -Uri "https://get.helm.sh/helm-$helmVersion-windows-amd64.zip" -OutFile "$env:TEMP\helm.zip"

# Extract it
Expand-Archive -Path "$env:TEMP\helm.zip" -DestinationPath "$env:TEMP\helm" -Force

# Place the binary somewhere on PATH
Copy-Item "$env:TEMP\helm\windows-amd64\helm.exe" "C:\Windows\System32\helm.exe"

# Verify
helm version
```

---

## Step 6 — Add the Crossplane Helm Repository

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
```

---

## Step 7 — Install Crossplane via Helm

Installs the Crossplane core controllers into their own namespace (`crossplane-system`):

```bash
helm upgrade --install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace
```

Example output confirming a successful install:

```text
Release "crossplane" does not exist. Installing it now.
NAME: crossplane
LAST DEPLOYED: Fri Jul 31 14:42:45 2026
NAMESPACE: crossplane-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Release: crossplane

Chart Name: crossplane
Chart Description: Crossplane is an open source Kubernetes add-on that enables
platform teams to assemble infrastructure from multiple vendors, and expose
higher level self-service APIs for application teams to consume.
Chart Version: 2.3.4
Chart Application Version: 2.3.4
Kube Version: v1.35.6
```

> ✅ At this point Crossplane's core controllers are running. Next, we install a **Provider** so Crossplane knows how to talk to Azure.

---

## Step 8 — Install the Azure Provider

The Azure "family" provider is the parent package that exposes Azure-specific CRDs (Managed Resources) to Kubernetes.

```bash
cat > provider-family-azure.yaml <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-family-azure
spec:
  package: xpkg.upbound.io/upbound/provider-family-azure:v1
EOF
```

Apply it and confirm it installs healthily:

```bash
kubectl apply -f provider-family-azure.yaml
kubectl get providers
kubectl get pods -n crossplane-system
```

```text
NAME                    INSTALLED   HEALTHY   PACKAGE                                            AGE
provider-family-azure   True        True      xpkg.upbound.io/upbound/provider-family-azure:v1   2m50s
```

> ⚠️ **Note:** installing only the specific Azure sub-providers you actually need (e.g. `provider-azure-network`, `provider-azure-storage`) instead of the full family package reduces footprint and limits RBAC scope — recommended for production.

---

## Step 9 — Authenticate the Provider to Azure

### Option A — Service Principal (simpler, less secure — used in this lab)

Create a Service Principal with the required role/scope. **⚠️ `Owner` at subscription scope is broad — fine for a lab, but scope it down (e.g. to a single resource group) for production.**

```bash
az ad sp create-for-rbac \
  --role Owner \
  --scopes "/subscriptions/0f45888e-fb6b-462e-b795-689434a41c26"
```

Inspect the generated credentials JSON:

```bash
cat azure-credentials.json
```

Store the credentials as a Kubernetes Secret so Crossplane's Azure provider can read them:

```bash
kubectl create secret generic azure-secret \
  -n crossplane-system \
  --from-file=creds=./azure-credentials.json
```

Create a **ProviderConfig** that tells the provider to authenticate using that secret:

```bash
cat > ProviderConfig.yaml <<EOF
apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-secret
      key: creds
EOF
```

```bash
kubectl apply -f ProviderConfig.yaml
```

> 🔐 **Better for production:** use **Azure Workload Identity** instead of a static Service Principal secret — it removes long-lived credentials from the cluster entirely. See the companion `crossplane-aks-guide.md` for the full Workload Identity steps.

---

## Step 10 — Provision an Azure Resource to Test

Define a simple `ResourceGroup` Managed Resource as a smoke test:

```bash
cat > ResourceGroup.yaml <<EOF
apiVersion: azure.upbound.io/v1beta1
kind: ResourceGroup
metadata:
  name: crossplane-test-rg
spec:
  forProvider:
    location: eastus
  providerConfigRef:
    name: default
EOF
```

Apply it:

```bash
kubectl apply -f ResourceGroup.yaml
```

Once applied, Crossplane's controller reconciles this manifest into a real Azure Resource Group. Check status with:

```bash
kubectl get resourcegroup.azure.upbound.io
```

If it shows `READY: True` and `SYNCED: True`, Crossplane successfully created the resource in Azure — confirming the full chain (Helm install → Provider → ProviderConfig → Managed Resource) is working end-to-end.

---

## Step 11 — Proving Reconciliation: Manually Deleting the Resource Group

Creating a resource is only half of what makes Crossplane interesting — the real value is **continuous reconciliation**: Crossplane doesn't just create a resource once and walk away, it keeps watching the live cloud state and corrects any drift so it always matches the desired state declared in the `ResourceGroup.yaml` manifest.

To actually prove this (not just assume it), the test performed here was:

1. Let Crossplane create `crossplane-test-rg` as shown in Step 10.
2. **Manually delete `crossplane-test-rg` from the Azure Portal** — simulating "drift" (someone/something changing infrastructure outside of Crossplane/GitOps).
3. Watch whether Crossplane notices the resource is gone and recreates it on its own, without any `kubectl apply` being re-run.

### What the Azure Activity Log shows

Checking **Azure Portal → Resource Group → Activity Log** after the manual delete shows two back-to-back events:

**Event 1 — the manual delete:**

```text
Operation: Delete resource group
Status: Succeeded
Initiated by: ayyppan1717@gmail.com   ← your own Azure account
Time: 9 minutes ago
```

This confirms the deletion really was done manually, by a human user, through the Portal — not by Crossplane.

**Event 2 — an automatic recreate, a few minutes later:**

```text
Operation: Update resource group
Status: Started
Initiated by: azure-cli-2026-07-31-...   ← NOT your user account
Time: 6 minutes ago
```

The key detail is the `Initiated by` field. It's **not** `ayyppan1717@gmail.com` this time — it's an identity named `azure-cli-2026-07-31-...`. That's the **Service Principal** created earlier in Step 9 with:

```bash
az ad sp create-for-rbac \
  --role Owner \
  --scopes "/subscriptions/0f45888e-fb6b-462e-b795-689434a41c26"
```

...i.e. the exact identity that Crossplane's Azure provider authenticates with via the `azure-secret` / `ProviderConfig`. So the recreate event was triggered by **Crossplane itself**, not by any manual action.

### The reconciliation loop, visually

```text
ResourceGroup YAML exists in Kubernetes (desired state)
        │
        ▼
Crossplane creates the RG in Azure
        │
        ▼
You manually delete the RG in the Azure Portal
        │
        ▼
Azure Activity Log records the deletion
        │
        ▼
Crossplane's controller notices actual state ≠ desired state ("drift")
        │
        ▼
Crossplane recreates the RG using its Service Principal
        │
        ▼
Activity Log shows a new Create/Update event from the azure-cli service principal
```

### Additional confirmation

The Kubernetes-side object never stopped reporting healthy, because from Crossplane's point of view it simply corrected drift — it never considered the resource "deleted":

```bash
kubectl get resourcegroup.azure.upbound.io
```

```text
NAME                 SYNCED   READY   AGE
crossplane-test-rg   True     True    2h
```

And on the Azure side, the resource group is provably back:

```bash
az group show --name crossplane-test-rg
```

```json
{
  "name": "crossplane-test-rg",
  "provisioningState": "Succeeded"
}
```

### 📸 Evidence — screenshots from this test

**1. Applying the manifest (initial creation)** — `kubectl apply -f ResourceGroup.yaml` submits the desired state to the Kubernetes API server, which triggers Crossplane's Azure provider to create the Resource Group for the first time.

<img width="840" height="150" alt="kubectl apply output for ResourceGroup" src="https://github.com/user-attachments/assets/8b429b80-0f6c-4d3a-9be9-eb76cace1f3e" />

**2. `SYNCED`/`READY` status in the cluster** — the Managed Resource reporting healthy in Kubernetes, both right after creation and again after Crossplane silently recreated it post-deletion.

<img width="1081" height="265" alt="kubectl get resourcegroup status" src="https://github.com/user-attachments/assets/5b45df5d-571b-4660-8541-191109d03358" />

**3. Azure Activity Log — the drift-and-recreate sequence** — this is the key evidence: a `Delete resource group` event initiated by the human Azure account, immediately followed by an `Update resource group` event initiated by the `azure-cli-...` Service Principal (Crossplane's identity), proving Crossplane detected the deletion and recreated the resource on its own.

<img width="1089" height="306" alt="Azure Activity Log showing delete then recreate by service principal" src="https://github.com/user-attachments/assets/34ec94ae-d1e8-45bf-b9f1-6f7345f4a0d1" />

**4. Confirmation in the Azure Portal** — `crossplane-test-rg` present and healthy in Azure again, despite having been manually deleted moments earlier — with no `kubectl apply` re-run by hand.

<img width="1032" height="259" alt="resource group visible again in Azure Portal after auto-recreation" src="https://github.com/user-attachments/assets/c56ca5b1-e340-47a2-9792-0f613b03f534" />

### ✅ Conclusion of the drift test

- The Resource Group was deleted manually via the Azure Portal.
- Crossplane's controller detected the drift (actual state no longer matched the `ResourceGroup` manifest's desired state).
- Crossplane automatically recreated the Resource Group — using the Service Principal configured in `ProviderConfig`/`azure-secret` — with no manual re-apply.
- This confirms the **Provider, ProviderConfig, Secret, and reconciliation loop are all correctly wired**, and demonstrates Crossplane's core differentiator vs. one-shot tools like Terraform: it doesn't just provision once, it continuously enforces the declared state.

> This is a good talking point for a demo/interview: *"I verified Crossplane's reconciliation by deleting an Azure Resource Group it had created. Crossplane detected the drift via its control loop and automatically recreated the Resource Group to match the desired state defined in Kubernetes — with no manual intervention."*

---

## ✅ Summary of What This Lab Achieved

1. Installed Azure CLI and authenticated to Azure.
2. Created an AKS cluster and connected `kubectl` to it.
3. Installed Helm and used it to deploy Crossplane's core controllers.
4. Installed the Azure Provider so Crossplane could speak the Azure API.
5. Authenticated the provider via a Service Principal secret.
6. Declared and provisioned a real Azure Resource Group purely through a Kubernetes manifest.
7. **Proved continuous reconciliation is actually working** — manually deleted the Resource Group in the Azure Portal and confirmed, via the Azure Activity Log, that Crossplane's controller detected the drift and recreated it automatically using its Service Principal, with no manual `kubectl apply`. This is Crossplane's core differentiator from one-shot IaC tools like Terraform: **infrastructure as Kubernetes-native, continuously reconciled objects.**

---

## 🔗 Next Steps / Further Reading

- Harden auth using **Workload Identity** instead of static Service Principal secrets.
- Scope the Service Principal role down from `Owner` to least-privilege.
- Explore **Compositions & XRDs** to expose a simplified, self-service `AKSCluster` or `Database` API to application teams.
- See the companion guide `crossplane-aks-guide.md` in this repo for: Crossplane vs. Terraform/Pulumi/ASO/KCC comparison, common operational challenges, security hardening checklist, and further blog/reference links.
