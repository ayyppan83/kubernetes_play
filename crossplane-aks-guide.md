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

# 8. Installing Crossplane on AKS — Step by Step

## Prerequisites

1. **An AKS cluster** already provisioned (or access to create one) — version 1.16+ (any modern supported AKS version works).
2. **kubectl** installed and configured with a context pointing at your AKS cluster (`az aks get-credentials`).
3. **Helm v3** installed.
4. **Azure CLI (`az`)** installed and logged in (`az login`), with sufficient permissions (Owner/Contributor) on the target subscription.
5. An **Azure Service Principal** (or Workload Identity setup — recommended, see Security section) for Crossplane's Azure provider to authenticate against Azure APIs.
6. Sufficient **cluster-admin RBAC** on the AKS cluster to install CRDs and cluster-scoped resources.
7. Outbound network access from the AKS nodes to `xpkg.crossplane.io` (or your private registry) to pull provider/function OCI packages.

## Step 1 — Connect kubectl to your AKS cluster

```bash
az aks get-credentials --resource-group <resource-group> --name <cluster-name>
kubectl get nodes   # sanity check
```

## Step 2 — Add the Crossplane Helm repository

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
```

## Step 3 — Install Crossplane via Helm

```bash
helm upgrade --install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace
```

## Step 4 — Verify Crossplane is running

```bash
kubectl get pods -n crossplane-system

kubectl wait --namespace crossplane-system \
  --for=condition=Ready pods \
  --all \
  --timeout=180s
```

## Step 5 — Install the Azure Provider

```bash
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure
spec:
  package: xpkg.crossplane.io/crossplane-contrib/provider-azure:v1.11.2
EOF

kubectl get providers
kubectl get pods -n crossplane-system   # provider pod should appear and become Ready
```

> Tip: Install only the specific Azure "family" providers you need (e.g., `provider-azure-network`, `provider-azure-storage`, `provider-azure-containerservice`) instead of the full monolithic provider, to reduce footprint and RBAC scope.

## Step 6 — Authenticate the Provider to Azure

### Option A: Service Principal (simpler, less secure)

```bash
az ad sp create-for-rbac --sdk-auth \
  --role Owner \
  --scopes /subscriptions/<subscription-id> \
  > azure-credentials.json

kubectl create secret generic azure-secret \
  -n crossplane-system \
  --from-file=creds=./azure-credentials.json
```

Create a `ProviderConfig` referencing the secret:

```yaml
apiVersion: azure.crossplane.io/v1beta1
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
```

```bash
kubectl apply -f providerconfig.yaml
```

### Option B: Workload Identity (recommended — no static secrets)

```bash
# Enable OIDC issuer + workload identity on the AKS cluster
az aks update \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --enable-oidc-issuer \
  --enable-workload-identity

export AKS_OIDC_ISSUER=$(az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

# Create a federated credential trusting Crossplane's service account
az identity federated-credential create \
  --name crossplane-federated-credential \
  --identity-name <managed-identity-name> \
  --resource-group <resource-group> \
  --issuer $AKS_OIDC_ISSUER \
  --subject system:serviceaccount:crossplane-system:crossplane \
  --audience api://AzureADTokenExchange
```

Then configure the `ProviderConfig` to use identity-based credentials instead of a static secret.

## Step 7 — Provision an Azure resource to test

```yaml
apiVersion: azure.crossplane.io/v1beta1
kind: ResourceGroup
metadata:
  name: crossplane-test-rg
spec:
  location: eastus
  providerConfigRef:
    name: default
```

```bash
kubectl apply -f test-rg.yaml
kubectl get resourcegroup.azure.crossplane.io
```

If the resource shows `READY: True` and `SYNCED: True`, Crossplane is correctly provisioning Azure resources.

## Step 8 (Optional) — Build a Composite Resource Definition (XRD) + Composition

Once basic MRs work, define reusable higher-level APIs (e.g., an `AKSCluster` XR bundling a VNet + Subnet + AKS cluster) so application teams can request infra via simple **Claims** rather than raw manifests.

---

## 9. Common Challenges When Installing Crossplane on AKS

- **Provider/package version compatibility** — mismatched Crossplane core vs. provider versions can cause subtle bugs (e.g., sequential instead of parallel resource creation, as seen with older Azure provider versions before v2.1.3/v2.2.0).
- **RBAC complexity** — Crossplane and its providers need broad cluster permissions to manage CRDs; scoping this down for least privilege takes careful planning.
- **Package registry access** — AKS nodes must reach `xpkg.crossplane.io` (or a private mirror/ACR); restrictive egress firewalls/NSGs can block provider pulls.
- **Static credential sprawl** — using Service Principal JSON secrets means long-lived credentials sitting in cluster secrets; rotating them is manual unless you move to Workload Identity.
- **Composition debugging** — patch-and-transform (or Function-based) Compositions can be hard to troubleshoot; errors often surface deep in controller logs rather than at `kubectl apply` time.
- **Resource deletion ordering** — while Crossplane handles dependency-aware deletion (e.g., waiting for VNets/Subnets to be removed before deleting a Resource Group), unexpected finalizer stalls can occur if a resource is externally deleted out-of-band.
- **State reconciliation drift with existing "ClickOps" resources** — importing/adopting pre-existing Azure resources into Crossplane management can be non-trivial.
- **Learning curve for platform teams** — designing good XRDs/Compositions requires platform-engineering maturity; poor abstractions can leak complexity back to developers.

## 10. Security Concerns

- **Least-privilege Azure roles** — avoid granting the Service Principal `Owner` at subscription scope (common in quick-start tutorials); scope it to specific resource groups and use custom roles with only the required permissions.
- **Prefer Workload Identity / Managed Identity over static Service Principal secrets** — eliminates long-lived credentials stored as Kubernetes Secrets, reducing exposure if the cluster is compromised.
- **Secrets at rest** — enable **encryption at rest** for etcd/Kubernetes Secrets in AKS, and consider Azure Key Vault CSI driver instead of storing raw credential JSON in a Secret object.
- **RBAC on Crossplane's own CRDs** — since any user who can create a Managed Resource or Claim can effectively provision cloud infrastructure, apply strict Kubernetes RBAC on who can create/edit `Provider`, `ProviderConfig`, MRs, and Claims.
- **Provider package supply-chain trust** — only pull provider/function OCI packages from trusted registries; verify signatures/checksums where available, and pin specific versions rather than `latest`.
- **Namespace isolation** — keep `crossplane-system` and provider credentials isolated from application namespaces; avoid granting broad namespace access to application teams.
- **Audit logging** — enable Kubernetes audit logs and Azure Activity Logs together so changes made via Crossplane are traceable both in-cluster and in Azure.
- **Blast radius of a single control plane** — because Crossplane can manage many cloud accounts/subscriptions from one cluster, a compromised cluster is a high-value target; treat the `crossplane-system` namespace with the same rigor as a CI/CD credential store.

---

## 11. Blogs / References

- Crossplane Official Documentation — concepts, API reference, install guides: https://docs.crossplane.io
- Crossplane with Workload Identity guide (official docs): https://docs.crossplane.io/latest/guides/crossplane-with-workload-identity/
- "Provision an AKS Cluster with Crossplane: A GitOps-Driven Approach to Azure Infrastructure" — Medium, Young Gyu Kim
- "A Complete Guide to Deploy Main Services in Azure with Crossplane" — Medium, Warley's CatOps
- "Crossplane with Azure AKS" — Medium, Harsha Rachith
- "Parallel AKS Node Pool Creation with Crossplane: A Version Compatibility Journey" — Microsoft Tech Community blog
- "Crossplane AWS Tutorial 2026: Kubernetes-Native Infrastructure Provisioning" — for cross-cloud comparison
- Upbound Marketplace (provider registry): https://marketplace.upbound.io
- CNCF Crossplane Project Page (governance, graduation info): https://www.cncf.io/projects/crossplane/
- Crossplane GitHub Repository: https://github.com/crossplane/crossplane
- provider-azure / provider-upjet-azure GitHub repos for provider-specific CRD references

---

*Note: Provider package versions (e.g., `provider-azure:v1.11.2`) and CLI flags shown above are illustrative — always check the Crossplane docs and Upbound Marketplace for the latest supported versions before installing in production.*
