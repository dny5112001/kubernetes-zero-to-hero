# 17 · Running Kubernetes in production

Everything so far works on a laptop. Production adds a few more jobs.

## 1. Use a managed cluster

Running the control plane yourself (etcd backups, upgrades, certificates) is
hard. Clouds do it for you:

| Cloud | Service |
|---|---|
| AWS | **EKS** (Elastic Kubernetes Service) |
| Google Cloud | **GKE** (Autopilot mode even manages the nodes) |
| Azure | **AKS** |
| Smaller/cheaper | DigitalOcean, Linode, Civo; **k3s** for small self-hosted setups |

Create the cluster itself with **Terraform** (e.g. the `terraform-aws-modules/eks`
module), not by clicking — see [terraform-zero-to-hero](https://github.com/dny5112001/terraform-zero-to-hero).

## 2. GitOps — git is the source of truth

Instead of running `kubectl apply` by hand or from CI, a tool **inside** the
cluster watches a git repo and keeps the cluster matching it:

```
you ──PR──► git repo (YAML / Helm / Kustomize) ◄──watches── Argo CD or Flux ──applies──► cluster
```

- Every change is a reviewed pull request.
- **Drift is fixed automatically** — someone `kubectl edit`s prod, it's put back.
- Rollback = `git revert`.
- Nobody needs write access to production clusters.

Tools: **Argo CD** (has a nice web UI) and **Flux**.

## 3. Observability — knowing what's going on

| Signal | Question | Common tools |
|---|---|---|
| **Metrics** | "How busy/slow/full is it?" | Prometheus + Grafana (often via `kube-prometheus-stack`) |
| **Logs** | "What happened?" | Loki, Elasticsearch/OpenSearch, CloudWatch |
| **Traces** | "Where did this request spend its time?" | OpenTelemetry + Tempo/Jaeger |
| **Alerts** | "Wake me if users are affected" | Alertmanager, PagerDuty |

Watch the basics for every app: request rate, error rate, response time
(the "RED" method), plus restarts and OOMKills.

## 4. Reliability checklist for every app

- [ ] At least **2 replicas** (3 for important things), spread across nodes/zones (`topologySpreadConstraints`).
- [ ] **Readiness and liveness probes** (chapter 11).
- [ ] **Resource requests** (and memory limits).
- [ ] A **PodDisruptionBudget** (chapter 13), so upgrades don't take it all down.
- [ ] **Graceful shutdown**: the app finishes in-flight requests on `SIGTERM` (`terminationGracePeriodSeconds`).
- [ ] An **HPA** if load varies.
- [ ] Images **pinned** by version, **scanned**, from your registry.
- [ ] **Secrets** from an external store, not in git.
- [ ] Runs as **non-root** with a locked-down securityContext (chapter 14).
- [ ] **NetworkPolicies** in place.
- [ ] **Backups** for anything with data (Velero for volumes; managed DB backups).

## 5. Upgrades

Kubernetes releases a new minor version roughly **3 times a year**, and each is
supported for about a year. You **must** upgrade regularly:

1. Read the release notes for removed APIs (tools: `kubent`, `pluto`).
2. Upgrade the **control plane** one minor version at a time (1.34 → 1.35, never skipping).
3. Upgrade **nodes** (drain, replace).
4. Upgrade add-ons (ingress controller, CNI, metrics-server).

Practise on a non-production cluster first.

## 6. Cost

- Requests decide how many nodes you pay for — set them from real usage (`kubectl top`, VPA recommendations).
- Use a node autoscaler (**Karpenter** on AWS, Cluster Autoscaler elsewhere) and spot instances for tolerant workloads.
- Tools like **OpenCost** / **Kubecost** show cost per namespace/team.

## 7. Multi-environment layout (a common setup)

```
clusters:  dev-cluster    staging-cluster    prod-cluster
           (Terraform)    (Terraform)        (Terraform)
apps:      git repo → Kustomize overlays/Helm values per env → Argo CD per cluster
```

## Remember

- Managed control plane, created with Terraform.
- GitOps (Argo CD / Flux) so git is the truth and drift self-heals.
- Metrics, logs, traces and alerts from day one.
- Go through the reliability checklist for every app; upgrade Kubernetes regularly.
