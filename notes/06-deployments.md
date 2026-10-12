# 06 · Deployments — many copies, self-healing, safe updates

A **Deployment** is how you run an app in real life. You say "keep 3 copies
of this Pod running", and it:
- creates them,
- **replaces** any that die,
- **updates** them to a new version a few at a time,
- lets you **roll back** if the new version is bad.

## The file

[`examples/06-deployments/deployment.yaml`](../examples/06-deployments/deployment.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 3                  # how many copies
  selector:
    matchLabels:
      app: whoami              # "the Pods I own have this label"
  template:                    # the Pod "cookie cutter"
    metadata:
      labels:
        app: whoami            # must match the selector above
    spec:
      containers:
        - name: whoami
          image: traefik/whoami:v1.11
          ports:
            - containerPort: 80
```

The `template` is a normal Pod description. The `selector` + labels are how
the Deployment recognises "its" Pods (more on labels in chapter 12).

```bash
kubectl apply -f examples/06-deployments/deployment.yaml
kubectl get deploy,rs,pods
```

## Three layers: Deployment → ReplicaSet → Pods

```
Deployment "whoami"            (manages versions and updates)
   └── ReplicaSet whoami-8d976fdbc   (keeps exactly N Pods of ONE version)
         ├── Pod whoami-8d976fdbc-xm75p
         ├── Pod whoami-8d976fdbc-szzsq
         └── Pod whoami-8d976fdbc-7g7qr
```

You almost never touch ReplicaSets yourself; the Deployment makes a new one
for each version.

## Self-healing — see it

```bash
kubectl delete pod <one-of-the-whoami-pods>
kubectl get pods        # a new one is already starting — the count goes back to 3
```

## Scaling

```bash
kubectl scale deploy/whoami --replicas=5
```

Better: change `replicas:` in the file and `kubectl apply` again, so the file
stays the truth. (Or let it scale automatically — chapter 11.)

## Rolling updates — new version, no downtime

Change the image:

```bash
kubectl set image deploy/whoami whoami=traefik/whoami:v1.10
kubectl rollout status deploy/whoami
```

What happens: Kubernetes starts a new Pod with the new version, waits until it's
**Ready**, then removes an old one — and repeats. Users keep being served the
whole time.

You control the pace:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # at most 1 extra Pod during the update
      maxUnavailable: 0    # never go below the wanted number
```

The other strategy, `Recreate`, stops all old Pods first, then starts new ones
(downtime, but never two versions at once).

> ⚠️ Kubernetes only knows a new Pod is "good" through its **readiness check**
> (chapter 11). Without one, a broken version can roll out completely.

## Rollback — undo a bad release

```bash
kubectl rollout history deploy/whoami     # list of versions (revisions)
kubectl rollout undo deploy/whoami        # back to the previous one
kubectl rollout undo deploy/whoami --to-revision=1
kubectl rollout restart deploy/whoami     # restart all Pods (e.g. to reload config)
```

## Try it

1. Apply the Deployment. Delete a Pod; watch it come back (`kubectl get pods -w`).
2. Scale to 5, then back to 3.
3. `kubectl set image ... v1.10`, watch the rollout, then `rollout undo`.
   Check the image: `kubectl get deploy whoami -o jsonpath='{.spec.template.spec.containers[0].image}'` → back to `v1.11`.
4. Set a typo image (`traefik/whoami:v9.99`) and watch: new Pods are stuck in
   `ImagePullBackOff`, but the **old Pods keep running** — the update stalls safely. Undo it.

## Remember

- Deployment → ReplicaSet → Pods. You manage the Deployment.
- It replaces dead Pods, scales, rolls out updates gradually, and rolls back.
- Rolling updates are only as safe as your readiness check.
