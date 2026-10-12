# 16 · Debugging — a calm, step-by-step method

Kubernetes problems feel scary because there are many layers. Follow the same
steps every time and you'll find almost everything.

## The method: look from the outside in

```
1. kubectl get pods            → What STATE is it in?
2. kubectl describe pod X      → Read the EVENTS at the bottom. What does Kubernetes say?
3. kubectl logs X [--previous] → What does the APP say?
4. kubectl exec / debug        → Look INSIDE.
5. Check the path              → Service selector, endpoints, ingress, network policy.
```

## Step 1 — the STATUS tells you where to look

| Status | Usually means | Look at |
|---|---|---|
| `Pending` | can't be placed on a node | `describe` → Events: "Insufficient memory/cpu", "didn't match node selector", "unbound PVC" |
| `ContainerCreating` (for long) | volume or Secret/ConfigMap missing | `describe` → Events |
| `ImagePullBackOff` / `ErrImagePull` | can't download the image | the image name/tag; registry login (`imagePullSecrets`) |
| `CrashLoopBackOff` | app starts then exits, over and over | `logs --previous` |
| `Error` | the container exited with an error | `logs` |
| `OOMKilled` | used more memory than its limit | raise the limit or fix the leak |
| `Running` but `0/1` Ready | readiness probe failing | `describe` → probe failures; the app's health endpoint |
| `Running`, `1/1`, but no traffic | the path to it is wrong | Step 5 |

## Step 2 — `describe` and Events

```bash
kubectl describe pod my-pod
kubectl get events --sort-by=.lastTimestamp          # everything recent in the namespace
kubectl get events --field-selector involvedObject.name=my-pod
```

The **Events** section is the single most useful debugging tool. Read it first.

## Step 3 — logs

```bash
kubectl logs my-pod
kubectl logs my-pod -c sidecar          # a specific container
kubectl logs my-pod --previous          # the run BEFORE the last crash — essential for CrashLoopBackOff
kubectl logs deploy/my-app              # one Pod of a Deployment
kubectl logs -l app=my-app --tail=50    # several Pods by label
```

## Step 4 — look inside

```bash
kubectl exec -it my-pod -- sh                                   # if the image has a shell
kubectl debug -it my-pod --image=busybox:1.37 --target=app      # attach a toolbox container (works even with no shell)
kubectl run tmp --rm -it --image=busybox:1.37 -- sh             # a test Pod: try wget/nslookup from inside the cluster
kubectl port-forward pod/my-pod 8095:80                         # test it from your laptop
```

## Step 5 — "it's running but I can't reach it"

Check the chain from the outside in:

```bash
kubectl get ingress                     # right host/path? right Service name?
kubectl get svc my-svc                  # right port / targetPort?
kubectl get endpointslices -l kubernetes.io/service-name=my-svc   # EMPTY = the selector matches no Ready Pods
kubectl get pods --show-labels          # do the labels match the Service selector exactly?
kubectl get networkpolicy               # is a policy blocking it?
```

An empty endpoint list is the classic: **the Service selector doesn't match
the Pod labels**, or the Pods aren't Ready.

## Practice: fix three broken Pods

```bash
kubectl apply -f examples/16-debugging/broken.yaml
kubectl get pods
# broken-one     0/1   ImagePullBackOff
# broken-two     0/1   CrashLoopBackOff (or Error)
# broken-three   0/1   Pending
```

Use the method to find out why each one is broken **before** reading the answers.

<details>
<summary>Answers</summary>

- **broken-one** — `describe` Events: `Failed to pull image "nginx:1.29-alpne"... not found`.
  A typo: `alpne` → `alpine`.
- **broken-two** — `kubectl logs broken-two --previous`:
  `cat: can't open '/config/settings.txt': No such file or directory`. The app
  expects a config file that was never mounted (it needs a ConfigMap volume), so it exits with an error and restarts forever.
- **broken-three** — `describe` Events:
  `0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 Insufficient memory.`
  It requests `500Gi` of memory. The two workers are too small, and the
  control-plane node is off-limits (its taint, chapter 13), so it can never be placed.
</details>

Clean up: `kubectl delete -f examples/16-debugging/broken.yaml`

## Node-level problems

```bash
kubectl get nodes                          # NotReady?
kubectl describe node <name>               # Conditions: MemoryPressure, DiskPressure...
kubectl top nodes                          # (needs metrics-server)
```

## Remember

- **get → describe (Events!) → logs (--previous) → exec/debug → check the path.**
- `Pending` = placement, `ImagePullBackOff` = image, `CrashLoopBackOff` = the app, `0/1 Ready` = probe.
- Empty endpoints = selector/labels mismatch or Pods not Ready.
