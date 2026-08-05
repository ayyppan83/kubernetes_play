# Interview Prep — Cloud/Kubernetes Platform Engineer

Based on the JD, this role blends **Platform Engineering + SRE + DevOps**. Expect four question tracks: Kubernetes depth, Cloud (Azure/AWS), GitOps/CI-CD/IaC, and Security/Observability/Ops maturity — plus scenario/troubleshooting questions and behavioral ones.

---

## 1. Kubernetes — Architecture & Administration

**Likely questions:**
- Walk me through what happens when you run `kubectl apply -f deployment.yaml` — from API call to pod running on a node.
- Explain the roles of kube-apiserver, etcd, scheduler, controller-manager, kubelet, kube-proxy.
- How does the scheduler decide where to place a pod? (affinity/anti-affinity, taints/tolerations, resource requests, topology spread constraints)
- Difference between Deployment, StatefulSet, DaemonSet — when to use each.
- How does a Service route traffic to pods? Difference between ClusterIP, NodePort, LoadBalancer, ExternalName.
- What's the difference between liveness, readiness, and startup probes? What happens if you misconfigure one?
- How do you do zero-downtime rolling updates? What's `maxSurge`/`maxUnavailable`?
- How do you upgrade a production cluster (control plane + node pools) with minimal disruption?

**Suggested prep approach:**
- Be ready to **draw the control plane diagram** on a whiteboard/screen-share.
- Have 1–2 real incidents ready where a probe misconfiguration or scheduling issue caused an outage, and how you fixed it.

---

## 2. RBAC, Secrets, Ingress

**Likely questions:**
- Explain Role vs ClusterRole, RoleBinding vs ClusterRoleBinding with an example.
- How would you design least-privilege RBAC for 3 teams sharing one cluster with namespace isolation?
- How do you manage secrets — native Kubernetes Secrets vs External Secrets Operator vs Azure Key Vault/AWS Secrets Manager/HashiCorp Vault integration? What are the weaknesses of native Secrets (base64, not encrypted at rest by default)?
- How do you enable **encryption at rest** for etcd secrets?
- Ingress controller options (NGINX, Traefik, AGIC on Azure, ALB Ingress Controller on AWS) — pros/cons and how you'd choose.
- How do you handle TLS termination and cert rotation at scale (cert-manager + Let's Encrypt or private CA)?
- How do you restrict egress/ingress traffic between namespaces? (NetworkPolicy, Calico/Cilium)

**Suggested prep approach:**
- Know **one concrete RBAC YAML** and **one NetworkPolicy YAML** cold — be ready to write them live.
- Be able to explain your secret-management architecture end-to-end (who can read what, rotation, audit trail).

---

## 3. Cloud Platforms (Azure and/or AWS)

**Likely questions:**
- Compare AKS vs EKS (or GKE) — managed control plane differences, node pool management, upgrade process, networking model (Azure CNI vs AWS VPC CNI).
- How do you design cluster networking — VNet/VPC peering, private clusters, private endpoints, hub-spoke topology?
- Explain IAM design: Azure AD Workload Identity / IRSA (IAM Roles for Service Accounts) for pod-to-cloud-resource auth without static credentials.
- How do you implement least-privilege IAM for CI/CD pipelines deploying to the cluster?
- What governance tools have you used — Azure Policy / AWS Organizations + SCPs, cost management, tagging standards?
- How do you handle multi-region / multi-subscription (Azure) or multi-account (AWS) landing zones?
- Explain your approach to network security groups / security groups, private link/private endpoint, and DNS design for private clusters.

**Suggested prep approach:**
- Pick **whichever cloud you're strongest in** and go deep — interviewers usually let you lead with your primary cloud, then ask you to map concepts to the other.
- Prepare a diagram of a landing-zone / hub-spoke network you've built or would build.

---

## 4. GitOps & CI/CD

**Likely questions:**
- Explain GitOps principles — declarative, versioned, pulled not pushed. How does ArgoCD/Flux differ from a traditional push-based CD pipeline?
- How do you structure a Git repo for multi-cluster, multi-environment GitOps (app-of-apps pattern, Kustomize overlays, Helm value files per env)?
- How do you handle secrets in a GitOps repo (Sealed Secrets, SOPS, External Secrets Operator)?
- Describe your CI/CD pipeline end to end: code commit → build → test → image scan → push to registry → deploy.
- How do you implement progressive delivery — canary or blue/green deployments (Argo Rollouts, Flagger)?
- How do you roll back a bad deployment safely and quickly?
- How do you manage drift between desired state (Git) and actual cluster state?

**Suggested prep approach:**
- Draw the GitOps reconciliation loop (Git → ArgoCD/Flux → cluster) and describe a rollback scenario end-to-end.
- Have a specific pipeline (tool names: GitHub Actions/Azure DevOps/GitLab CI/Jenkins + ArgoCD/Flux) you can describe stage by stage.

---

## 5. Infrastructure as Code (IaC)

**Likely questions:**
- Terraform vs Bicep/ARM vs Pulumi — trade-offs.
- How do you structure reusable Terraform modules (module registry, versioning, remote state, workspaces)?
- How do you manage Terraform state safely in a team (remote backend, state locking, drift detection)?
- How do you test IaC before merging (terraform plan in CI, `tflint`, `checkov`/`tfsec`, policy-as-code with OPA/Sentinel)?
- How would you design a self-service IaC platform so app teams can provision namespaces/resources without full cloud admin access?

**Suggested prep approach:**
- Know a real module you built (e.g., "reusable AKS cluster module with configurable node pools, RBAC, and network policy") and be ready to explain its interface (inputs/outputs) and versioning strategy.

---

## 6. Observability (Monitoring, Logging, Alerting)

**Likely questions:**
- Design a monitoring stack for a Kubernetes platform — Prometheus + Grafana + Alertmanager, or Azure Monitor/Container Insights, or AWS CloudWatch/Container Insights.
- How do you monitor at 3 layers — infrastructure (nodes), platform (control plane, etcd), application (pods, custom metrics)?
- Difference between metrics, logs, and traces — how do they complement each other (OpenTelemetry)?
- How do you avoid alert fatigue — SLOs/SLIs, error budgets, meaningful alert thresholds?
- How do you centralize logs across many clusters (Fluent Bit/Fluentd → Loki/ELK/Azure Log Analytics/CloudWatch Logs)?
- Describe an incident where observability tooling helped you find root cause quickly.

**Suggested prep approach:**
- Prepare one **SLO example** (e.g., "99.9% availability, error budget of X minutes/month") and how you'd alert on burn rate.

---

## 7. Security, Governance & Compliance

**Likely questions:**
- How do you secure the software supply chain — image scanning (Trivy/Grype), signed images (cosign/Sigstore), admission control (OPA Gatekeeper/Kyverno)?
- How do you enforce pod security standards (Pod Security Admission, restricted profile — no root, no privilege escalation, read-only root filesystem)?
- How do you handle compliance frameworks (SOC2, ISO 27001, CIS Benchmarks for Kubernetes) in a regulated environment?
- How do you audit who did what on the cluster (audit logs, SIEM integration)?
- How would you respond to a compromised pod / container escape incident?

**Suggested prep approach:**
- Know the **CIS Kubernetes Benchmark** at a high level and 2–3 concrete controls you've implemented (e.g., disabling anonymous API access, restricting hostPath mounts).

---

## 8. Scenario / Troubleshooting Questions (very likely to be asked live)

Be ready to talk through your **diagnostic process out loud**, not just the answer:

- "A pod is stuck in `CrashLoopBackOff` — walk me through your debugging steps." (`kubectl describe pod`, `kubectl logs --previous`, check resource limits, probes, image, config)
- "Users report the app is slow, but pods look healthy — what do you check?" (node resource pressure, HPA scaling lag, network latency, DNS resolution, downstream dependency, throttling)
- "A deployment update is stuck at 50% rollout — diagnose." (readiness probe failing, PodDisruptionBudget blocking, insufficient node capacity, image pull errors)
- "Production cluster is out of IP addresses in the VNet/VPC — what are your options?" (CNI overlay mode, secondary IP ranges, redesigning subnet sizing)
- "Cost has spiked 3x this month on the cluster — how do you investigate and fix?" (idle namespaces, missing resource requests/limits causing over-provisioning, orphaned PVs/load balancers, right-sizing node pools, Karpenter/cluster-autoscaler tuning)
- "etcd is under memory pressure / cluster is unresponsive — what do you do?" (check etcd disk latency, compact/defrag, review request rate, scale control plane if self-managed)

**Suggested prep approach:** Structure every answer as: **Observe → Isolate → Hypothesize → Verify → Fix → Prevent recurrence**. Interviewers reward a clear methodology more than a guessed right answer.

---

## 9. Platform Engineering / Developer Self-Service (a growing focus area)

- How would you build a self-service platform so developers can spin up environments without filing tickets? (Backstage.io portal, Crossplane, templated Helm charts, golden paths)
- How do you balance developer autonomy with governance guardrails?
- What does "platform as a product" mean to you?

---

## 9b. Crossplane (called out specifically — go deep here)

Crossplane turns your cloud/infra provisioning into Kubernetes-native APIs — it's a strong signal this role wants **platform engineering with Kubernetes as the control plane for infrastructure**, not just app deployment. Expect real depth here.

**Concepts to know cold:**
- **Providers** — install cloud API support (`provider-azure`, `provider-aws`, `provider-family-aws`, `provider-upjet-*`) that give you Managed Resources (MRs) as CRDs (e.g., `Cluster.azure.upbound.io`, `RDSInstance.aws.upbound.io`).
- **Managed Resources (MRs)** — the 1:1 Kubernetes CRD representation of a real cloud resource (a VNet, an S3 bucket, an AKS cluster). Reconciled continuously like any K8s controller — this is what gives Crossplane **drift correction** that plain Terraform doesn't have out of the box.
- **Compositions + XRDs (Composite Resource Definitions)** — how you build your own platform APIs. You define a custom abstraction (e.g., `XPostgresInstance`) and a Composition that maps it to a bundle of underlying MRs (network, subnet, instance, firewall rule). This is literally "build a self-service internal platform."
- **Claims** — the namespaced object developers actually request (e.g., `PostgresInstanceClaim`) — gives app teams a simple, governed self-service interface without touching cloud consoles or raw Terraform.
- **Composition Functions (`crossplane-function-*`, e.g. function-patch-and-transform, function-go-templating, function-kcl)** — the newer, more powerful way to write composition logic instead of the older patch-and-transform-only model. Worth knowing this evolved from "Compositions v1 (patch & transform only)" to "Functions pipeline" — if you've only used the old style, say so honestly and mention you're aware of the newer pattern.
- **ProviderConfig** — how Crossplane authenticates to the cloud (e.g., Azure Workload Identity / AWS IRSA bound to the Crossplane provider's ServiceAccount) — ties directly back to your IAM answer.
- **Crossplane vs Terraform vs ArgoCD** — be ready to articulate this clearly:
  - *Terraform*: run-based (apply once, no continuous reconciliation unless you re-run); state file lives outside the cluster.
  - *Crossplane*: control-loop based, continuous drift detection/correction, state lives as Kubernetes objects (`kubectl get` your infra), composable into GitOps naturally.
  - *ArgoCD/Flux*: deploys the Crossplane manifests (Claims/Compositions) themselves — Crossplane and GitOps are complementary, not competing (GitOps drives *what* Crossplane objects exist; Crossplane drives *how* they become real cloud resources).
- **Crossplane vs Terraform provider for Kubernetes/Helm** — some orgs use Terraform for foundational cloud (VPC/subscription-level) and Crossplane for the self-service layer on top; be ready to discuss where you'd draw that line.

**Likely interview questions:**
- "How would you design a self-service database-provisioning API for app teams using Crossplane?" — walk through: XRD → Composition (network + instance + secret output) → Claim → RBAC scoping the Claim to a namespace → secret auto-populated for the app to consume.
- "How do you handle secrets that Crossplane generates (e.g., a DB password) securely?" — Crossplane writes connection secrets as native K8s Secrets; pair with External Secrets Operator/Key Vault sync if you need them outside the cluster, and RBAC-restrict who can read the Secret namespace.
- "How do you version and roll out changes to a Composition without breaking existing Claims?" — Composition revisions, testing in lower environments, gradual migration strategy.
- "How do you handle multi-cloud abstraction?" — a single XRD/Claim API that different Compositions satisfy per provider/environment (e.g., same `XDatabase` claim resolves to Azure Flexible Server in one env, RDS in another) — this is Crossplane's core value pitch, be ready to explain it clearly.
- "What are the operational risks of Crossplane?" — provider CRD sprawl, upgrade coordination between core Crossplane and providers, RBAC complexity (who can create MRs directly vs only Claims), debugging composition failures (`kubectl describe` chains through XR → MR → provider events).

**If you haven't used Crossplane hands-on:** be honest, but bridge from what you do know — e.g., "I've built the equivalent self-service pattern with Terraform modules + a thin abstraction layer / Backstage templates, and I understand Crossplane's advantage is doing that natively as Kubernetes CRDs with continuous reconciliation rather than run-based apply." That shows you understand *why* they want it, even without direct hours on the tool. Consider spending 1–2 hours before the interview spinning up a local Crossplane instance (kind cluster + `provider-aws`/`provider-azure` + one simple Composition) so you can speak from firsthand, current experience.

---

## 10. Behavioral Questions (STAR format)

- Tell me about a production incident you led the response for.
- Describe a time you had to convince a team to adopt a new standard (e.g., moving from manual deploys to GitOps).
- Tell me about a time you disagreed with a security/compliance requirement — how did you resolve it?
- Describe a project where you reduced operational toil through automation.

Prepare 4–5 STAR stories in advance covering: an outage, a security incident, a cross-team collaboration, and a cost/efficiency win.

---

## 11. Questions You Should Ask Them

- What's the current state of the platform — greenfield, or migrating from something legacy?
- Single cluster or multi-cluster/multi-region? Multi-tenant or dedicated clusters per team?
- What's the GitOps/CI-CD toolchain today?
- How is the platform team organized relative to app teams — full platform-as-a-product, or shared ops?
- What does on-call look like for this role?

---

## Quick Prep Checklist

- [ ] Review core `kubectl` commands and troubleshooting flags (`describe`, `logs -p`, `get events --sort-by`, `top pod`)
- [ ] Write/review one Deployment + Service + Ingress + NetworkPolicy + RBAC YAML from memory
- [ ] Refresh Terraform module structure (variables, outputs, remote state)
- [ ] Refresh IAM model for your primary cloud (Workload Identity / IRSA)
- [ ] Prepare 2 architecture diagrams: cluster network topology, and GitOps CI/CD flow
- [ ] Prepare 4–5 STAR stories
- [ ] Review CIS Kubernetes Benchmark top controls
- [ ] Have 3–5 questions ready to ask the interviewer

---

*Tip: for a role this broad, interviewers often go deep on 2–3 areas rather than shallow across all of them. Lead with your strongest area (cloud, K8s, or CI/CD) early in your answers so the conversation gravitates there.*
