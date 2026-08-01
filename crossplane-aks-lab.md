# Crossplane on AKS — Lab Walkthrough

> A hands-on log of installing **Crossplane** on an **Azure Kubernetes Service (AKS)** cluster and using it to provision a real Azure resource (a Resource Group), with an overview of what Crossplane is and why it's used.

---

## 📖 What is Crossplane? (Quick Summary)

**Crossplane** is a **CNCF-graduated, open-source Kubernetes add-on** that turns a Kubernetes cluster into a **universal control plane** for cloud infrastructure. Instead of provisioning resources through cloud consoles, CLIs, or a separate Terraform/IaC pipeline, you describe infrastructure — VNets, databases, storage accounts, AKS clusters, DNS records, etc. — as Kubernetes YAML manifests (custom resources). Crossplane's controllers then continuously **reconcile** the real cloud resource to match that declared state, the same control-loop pattern Kubernetes already uses for Pods and Deployments.

In short: **`kubectl apply` for your cloud infrastructure, not just your containers.**

**Core building blocks:**

| Concept | What it does |
|---|---|
| **Provider** | A plugin that talks to a specific cloud/service API (Azure, AWS, GCP, GitHub, etc.) and exposes its resources as Kubernetes CRDs |
| **Managed Resource (MR)** | The lowest-level representation of one cloud resource (e.g. an Azure `ResourceGroup`) |
| **ProviderConfig** | Tells a Provider *how to authenticate* to the cloud (credentials/secret reference) |
| **Composition / XRD** | Lets platform teams bundle several MRs into one reusable, higher-level self-service API |
| **Claim** | The simple, self-service request an app team submits without needing to know cloud internals |

**Why it matters:** it unifies infrastructure and application delivery under one GitOps model, gives continuous drift correction (unlike one-shot `terraform apply`), and enables platform teams to offer safe, self-service infrastructure APIs to developers — across multiple clouds if needed.

> 📚 For a deeper dive (competitors, security concerns, common challenges, future roadmap), see the companion reference doc `crossplane-aks-guide.md`.

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

### 📸 Screenshots from this run

| Step | Screenshot |
|---|---|
| Resource applied | ![resource applied](https://github.com/user-attachments/assets/8b429b80-0f6c-4d3a-9be9-eb76cace1f3e) |
| Resource status in cluster | ![resource status](https://github.com/user-attachments/assets/5b45df5d-571b-4660-8541-191109d03358) |
| Resource visible in Azure Portal | ![azure portal](https://github.com/user-attachments/assets/34ec94ae-d1e8-45bf-b9f1-6f7345f4a0d1) |
| Final confirmation | ![final confirmation](https://github.com/user-attachments/assets/c56ca5b1-e340-47a2-9792-0f613b03f534) |

---

## ✅ Summary of What This Lab Achieved

1. Installed Azure CLI and authenticated to Azure.
2. Created an AKS cluster and connected `kubectl` to it.
3. Installed Helm and used it to deploy Crossplane's core controllers.
4. Installed the Azure Provider so Crossplane could speak the Azure API.
5. Authenticated the provider via a Service Principal secret.
6. Declared and provisioned a real Azure Resource Group purely through a Kubernetes manifest — proving Crossplane's core promise: **infrastructure as Kubernetes-native, continuously reconciled objects.**

---

## 🔗 Next Steps / Further Reading

- Harden auth using **Workload Identity** instead of static Service Principal secrets.
- Scope the Service Principal role down from `Owner` to least-privilege.
- Explore **Compositions & XRDs** to expose a simplified, self-service `AKSCluster` or `Database` API to application teams.
- See the companion guide `crossplane-aks-guide.md` in this repo for: Crossplane vs. Terraform/Pulumi/ASO/KCC comparison, common operational challenges, security hardening checklist, and further blog/reference links.
