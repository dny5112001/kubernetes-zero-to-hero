# 12 · Namespaces, labels and organising a cluster

## Namespaces — rooms in a house

A **namespace** divides one cluster into separate areas: per team, per app, or
per environment. Names only need to be unique **inside** a namespace, so two
teams can both have a Service called `api`.

```bash
kubectl get namespaces
# default          <- where things go if you don't say
# kube-system      <- Kubernetes' own parts
# kube-public, kube-node-lease  <- internal
```

```bash
kubectl create namespace team-a
kubectl apply -f app.yaml -n team-a
kubectl get pods -n team-a
kubectl get pods -A                                   # all namespaces
kubectl config set-context --current --namespace=team-a   # change your default
```

Namespaces give you a place to attach:
- **budgets** (ResourceQuota) and **defaults** (LimitRange) — below,
- **permissions** (RBAC, chapter 14),
- **network rules** (NetworkPolicy, chapter 14).

> ⚠️ A namespace is **not** a security wall by itself. Pods in different
> namespaces can still talk to each other unless you add NetworkPolicies.

Some things aren't in any namespace: Nodes, PersistentVolumes, StorageClasses,
Namespaces themselves. See which: `kubectl api-resources --namespaced=false`.

## Budgets and defaults

[`examples/12-namespaces/team-space.yaml`](../examples/12-namespaces/team-space.yaml)
creates `team-a` with:
- a **ResourceQuota** — the team's total budget (max 5 Pods, 1 CPU of requests...),
- a **LimitRange** — default requests/limits for containers that don't set any.

```bash
kubectl apply -f examples/12-namespaces/team-space.yaml
kubectl -n team-a create deployment many --image=busybox:1.37 --replicas=7 -- sleep 3600
kubectl -n team-a get deploy many            # READY 5/7 — the quota stopped 2
kubectl -n team-a get events | grep quota
# ... forbidden: exceeded quota: budget, requested: pods=1, used: pods=5, limited: pods=5
```

## Labels — sticky notes on objects

A **label** is a `key: value` sticker you put on anything:

```yaml
metadata:
  labels:
    app: shop
    tier: backend
    env: prod
```

Labels are how Kubernetes **groups** things. You've already used them:
- a **Deployment** finds its Pods by label (`selector.matchLabels`),
- a **Service** sends traffic to Pods by label (`selector`),
- a **NetworkPolicy** chooses Pods by label.

**Selectors** filter by labels:

```bash
kubectl get pods -l app=whoami
kubectl get pods -l 'env in (dev,staging)'
kubectl get pods -l app=whoami,tier!=frontend
kubectl get pods --show-labels
kubectl label pod hello team=blue            # add one
```

Recommended standard labels:

```yaml
app.kubernetes.io/name: shop
app.kubernetes.io/instance: shop-prod
app.kubernetes.io/version: "1.4.2"
app.kubernetes.io/component: backend
app.kubernetes.io/part-of: webstore
app.kubernetes.io/managed-by: helm
```

## Annotations — notes for tools, not for grouping

**Annotations** are also key/values, but you can't select by them. They hold
extra information for tools and humans:

```yaml
metadata:
  annotations:
    description: "Handles checkout"
    prometheus.io/scrape: "true"
```

Rule: **labels to find and group, annotations to describe and configure tools.**

## How to split a real cluster

| Approach | When |
|---|---|
| One namespace per **app/team** in a shared cluster | most common |
| One namespace per **environment** (dev, staging) in a non-prod cluster | small teams |
| **Separate clusters** for prod vs non-prod | strongly recommended — a mistake in dev can't touch prod |

## Remember

- Namespaces = rooms: separate names, budgets, permissions. Not a security wall alone.
- Labels = sticky notes used to **select** things. Annotations = notes for tools.
- ResourceQuota = team budget; LimitRange = sensible defaults.
