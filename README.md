# Kubernetes: Zero to Hero

Plain-English notes for learning Kubernetes from nothing to "I can run apps on
it at work". No jargon without an explanation. Every chapter has examples you
run on a **free local cluster on your laptop** — no cloud account needed.

## How to use these notes

1. Read a chapter (10–20 minutes each).
2. Run its example from `examples/` — and **break things on purpose**.
3. Do the "Try it" bits at the end of each chapter.
4. Then build the bigger project: [k8s-terraform-lab](https://github.com/dny5112001/k8s-terraform-lab)
   (Kubernetes + Terraform + fake AWS, all local). Learning Terraform too? See
   [terraform-zero-to-hero](https://github.com/dny5112001/terraform-zero-to-hero).

## Chapters

### Part 1 — Zero: the basics
| # | Chapter | You'll learn |
|---|---|---|
| 01 | [What is Kubernetes?](notes/01-what-is-kubernetes.md) | The problem it solves, in plain words |
| 02 | [Containers in 10 minutes](notes/02-containers-refresher.md) | Images, containers, registries — what Kubernetes runs |
| 03 | [Your own cluster + kubectl](notes/03-setup-and-kubectl.md) | kind, kubectl, YAML, `apply`/`get`/`describe` |
| 04 | [How Kubernetes works inside](notes/04-how-it-works.md) | Control plane, nodes, and the "desired state" loop |
| 05 | [Pods](notes/05-pods.md) | The smallest thing Kubernetes runs |

### Part 2 — Running real apps
| # | Chapter | You'll learn |
|---|---|---|
| 06 | [Deployments](notes/06-deployments.md) | Many copies, self-healing, zero-downtime updates, rollback |
| 07 | [Services](notes/07-services.md) | Stable addresses and load balancing; DNS inside the cluster |
| 08 | [Ingress and Gateway API](notes/08-ingress.md) | One front door for web traffic |
| 09 | [ConfigMaps and Secrets](notes/09-config-and-secrets.md) | Settings and passwords, outside the image |
| 10 | [Storage](notes/10-storage.md) | Disks that survive restarts; StatefulSets for databases |
| 11 | [Health checks, resources, autoscaling](notes/11-health-resources-autoscaling.md) | Probes, requests/limits, HPA |

### Part 3 — Hero: production Kubernetes
| # | Chapter | You'll learn |
|---|---|---|
| 12 | [Namespaces, labels and organising](notes/12-namespaces-labels.md) | Dividing a cluster; budgets per team |
| 13 | [Other workload types](notes/13-workload-types.md) | Jobs, CronJobs, DaemonSets — and where Pods get placed |
| 14 | [Security](notes/14-security.md) | RBAC, service accounts, locked-down Pods, network policies |
| 15 | [Packaging: Helm and Kustomize](notes/15-helm-and-kustomize.md) | Reusable, per-environment app setups |
| 16 | [Debugging](notes/16-debugging.md) | A step-by-step method + fixing 3 broken Pods |
| 17 | [Running it in production](notes/17-production.md) | Managed clusters, monitoring, GitOps, upgrades |
| 18 | [Cheat sheet](notes/18-cheat-sheet.md) | Every command on one page |
| 19 | [Glossary + interview questions](notes/19-glossary-and-interview.md) | Check yourself |

## Quick start

You need Docker Desktop (give it ≥ 4 GB RAM), `kubectl`, `kind`, and (from chapter 08) `helm`.

```bash
kind create cluster --config examples/kind-cluster.yaml   # 3-node cluster called "learn"
kubectl get nodes
```

When you're done for the day:

```bash
kind delete cluster --name learn    # removes everything; recreate any time
```

Ports used on your laptop: **8090** (ingress, ch. 08) and **8091** (NodePort, ch. 07).
