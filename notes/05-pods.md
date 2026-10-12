# 05 · Pods — the smallest thing Kubernetes runs

## What's a Pod?

Kubernetes doesn't run containers directly. It runs **Pods**. A Pod is a
**wrapper around one (or a few) containers** that always live together on the
same node.

Think of a Pod as a **small flat**: usually one person (container) lives there.
Sometimes two people share it — they share the **address** (IP) and can share
**cupboards** (volumes).

Containers in the same Pod:
- share one **IP address**, so they talk via `localhost`,
- can share **folders** (volumes),
- are **started, stopped and moved together**.

## Your first Pod

[`examples/05-pods/pod.yaml`](../examples/05-pods/pod.yaml):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
  labels:
    app: hello
spec:
  containers:
    - name: web
      image: nginx:1.29-alpine
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f examples/05-pods/pod.yaml
kubectl get pods
kubectl port-forward pod/hello 8095:80     # then open http://localhost:8095
```

`port-forward` makes a temporary tunnel from your laptop into the Pod —
perfect for quick testing. (Ctrl+C to stop.)

## A Pod's life

| Phase/status | Meaning |
|---|---|
| `Pending` | Accepted, but not running yet: waiting for a node, or pulling the image |
| `ContainerCreating` | Node chosen; pulling the image and starting |
| `Running` | At least one container is running |
| `Succeeded` | All containers finished OK (for one-off tasks) |
| `Failed` | Containers finished with an error |
| `CrashLoopBackOff` | It keeps crashing; Kubernetes waits longer between each restart |
| `ImagePullBackOff` | It can't download the image (typo, private registry, no internet) |

**Ready** (the `1/1` column) means the Pod passed its health check and can get
traffic (chapter 11).

## Two containers in one Pod (sidecar pattern)

[`examples/05-pods/pod-two-containers.yaml`](../examples/05-pods/pod-two-containers.yaml):
a `writer` container adds the date to a file every 5 seconds; an `nginx`
container serves that same file. They share an `emptyDir` volume.

```bash
kubectl apply -f examples/05-pods/pod-two-containers.yaml
kubectl exec two-containers -c web -- wget -qO- localhost
# Mon Oct 12 06:24:01 UTC 2026
# Mon Oct 12 06:24:06 UTC 2026
```

`-c web` picks which container when there's more than one.

Use multiple containers **only** when they really must live together: a helper
that ships logs, a proxy, a file syncer. **Different apps = different Pods.**

There are also **init containers**: they run *before* the main containers, one
after another, and must finish first (e.g. "wait until the database is up",
"download config"). And Kubernetes supports **native sidecars**: an init
container with `restartPolicy: Always` that starts first and keeps running
alongside the app.

## Don't create Pods by hand (in real life)

A plain Pod is **not looked after**. If its node dies, the Pod is gone for
good. Delete it, and it's gone. Try it:

```bash
kubectl delete pod hello
kubectl get pods     # gone, nothing brings it back
```

In real life you create a **Deployment** (next chapter), which creates and
**replaces** Pods for you. Plain Pods are for learning and quick debugging.

## Useful Pod commands

```bash
kubectl logs hello                 # output
kubectl logs hello -f              # follow live
kubectl logs hello --previous      # output from BEFORE the last crash (very useful!)
kubectl exec -it hello -- sh       # a shell inside
kubectl describe pod hello         # details + Events
kubectl get pod hello -o wide      # which node, which IP
kubectl run tmp --rm -it --image=busybox:1.37 -- sh   # a throwaway Pod to test things from inside
```

## Remember

- Pod = one or a few containers that live together, share an IP and can share folders.
- Plain Pods aren't replaced if they die — use Deployments.
- `logs`, `logs --previous`, `exec`, `describe`, `port-forward` are your friends.
