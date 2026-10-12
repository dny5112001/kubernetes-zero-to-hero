# 03 · Your own cluster + kubectl

## Install the tools (Mac)

```bash
brew install kubectl kind helm
```

- **kind** ("Kubernetes IN Docker") — makes a real Kubernetes cluster where each
  "computer" (node) is a Docker container. Free, fast, disposable.
- **kubectl** ("cube-control") — the remote control for any Kubernetes cluster.
- **helm** — an app installer for Kubernetes (chapter 15).

Other local options: minikube, k3d, Docker Desktop's built-in Kubernetes,
Rancher Desktop, OrbStack. They all work with the same `kubectl` commands.

## Make a cluster

```bash
kind create cluster --config examples/kind-cluster.yaml
kubectl get nodes
```

```
NAME                  STATUS   ROLES           AGE   VERSION
learn-control-plane   Ready    control-plane   1m    v1.35.8
learn-worker          Ready    <none>          1m    v1.35.8
learn-worker2         Ready    <none>          1m    v1.35.8
```

Three "computers": one **control plane** (the brain) and two **workers** (where
apps run). Chapter 04 explains them.

## How kubectl knows which cluster to talk to

kubectl reads `~/.kube/config` (the **kubeconfig**). It can hold many clusters.
Each cluster + user pair is a **context**.

```bash
kubectl config get-contexts           # list them (* = current)
kubectl config use-context kind-learn # switch
kubectl config current-context        # which one am I on? CHECK THIS before deleting things!
```

## Everything is described in YAML

You describe objects in YAML files. Every object has the same four parts:

```yaml
apiVersion: v1          # which version of the Kubernetes API this type belongs to
kind: Pod               # what type of thing
metadata:               # its name, labels, namespace
  name: hello
  labels:
    app: hello
spec:                   # what you WANT (the desired state)
  containers:
    - name: web
      image: nginx:1.29-alpine
```

After you create it, Kubernetes adds a **`status`** section: what's **actually**
happening. `spec` = wish, `status` = reality.

### YAML in 30 seconds

```yaml
key: value              # a setting
nested:                 # indentation (2 spaces, never tabs) = "inside"
  child: value
list:
  - first               # a dash = a list item
  - second
---                     # separates several objects in one file
```

## The kubectl commands you'll use every day

```bash
kubectl apply -f file.yaml      # create or update from a file (use this most)
kubectl apply -f folder/        # every YAML file in a folder
kubectl get pods                # list (also: deployments, services, nodes, all...)
kubectl get pods -o wide        # more columns (IP, node)
kubectl get pod hello -o yaml   # the full object, including status
kubectl describe pod hello      # human-friendly details + EVENTS (great for debugging)
kubectl logs hello              # what the container printed
kubectl exec -it hello -- sh    # open a shell inside the container
kubectl delete -f file.yaml     # delete what the file describes
kubectl explain pod.spec.containers   # built-in docs for any field
```

### `apply` vs `create`

- `kubectl create` = "make this; error if it exists" (a one-off order).
- `kubectl apply` = "make it look like this file" (declarative). Run it again
  after editing the file and it updates. **Prefer `apply`**, and keep your
  YAML files in git.

### Short names save typing

`po` = pods, `deploy` = deployments, `svc` = services, `ns` = namespaces,
`cm` = configmaps, `pvc` = persistentvolumeclaims, `no` = nodes.

```bash
kubectl get po,svc,deploy
```

Many people also `alias k=kubectl`.

## Try it

```bash
kubectl apply -f examples/05-pods/pod.yaml
kubectl get pods -w              # -w = watch live; Ctrl+C to stop
kubectl describe pod hello       # read the Events at the bottom
kubectl explain pod.spec         # browse the docs
kubectl delete -f examples/05-pods/pod.yaml
```

## Remember

- kind = a free, real cluster in Docker. kubectl = the remote control.
- Every object: `apiVersion`, `kind`, `metadata`, `spec` (+ `status` from Kubernetes).
- `apply -f`, `get`, `describe`, `logs`, `exec`, `delete` — and **check your context**.
