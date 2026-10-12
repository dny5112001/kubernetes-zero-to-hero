# 11 · Health checks, resources and autoscaling

Three things that separate "it runs" from "it runs reliably".

## Part 1 — Health checks (probes)

"Running" doesn't mean "working". The process might be up but stuck, or still
loading. **Probes** are questions the kubelet asks your app regularly.

| Probe | Question | If it fails |
|---|---|---|
| **startupProbe** | "Have you finished starting?" | keep waiting; the other probes don't run yet. Too long → restart |
| **readinessProbe** | "Can you take traffic **right now**?" | removed from the Service (no traffic), **not** restarted |
| **livenessProbe** | "Are you stuck/broken?" | container is **restarted** |

[`examples/11-health-and-resources/deployment.yaml`](../examples/11-health-and-resources/deployment.yaml):

```yaml
startupProbe:
  httpGet: { path: /health, port: 80 }
  failureThreshold: 10         # allow up to 10 × 2s = 20s to start
  periodSeconds: 2
readinessProbe:
  httpGet: { path: /health, port: 80 }
  periodSeconds: 5
livenessProbe:
  httpGet: { path: /health, port: 80 }
  periodSeconds: 10
  failureThreshold: 3          # 3 failures in a row → restart
```

Probe types: `httpGet` (a 200–399 answer = OK), `tcpSocket` (can I connect?),
`exec` (run a command; exit 0 = OK), `grpc`.

### The golden rules

- **Readiness** may check dependencies ("can I reach the database?").
- **Liveness must NOT check dependencies.** If the database goes down and
  liveness checks it, Kubernetes restarts *all* your app Pods in a loop — making
  everything worse. Liveness = "is *my process* healthy?" only.
- If your app starts slowly, use a **startupProbe** instead of a long delay on liveness.
- Readiness is what makes **rolling updates safe** (chapter 06).

## Part 2 — Resources: requests and limits

```yaml
resources:
  requests:            # "I need at least this" — the scheduler uses it to pick a node
    cpu: 50m
    memory: 32Mi
  limits:              # "never let me use more than this"
    memory: 64Mi
```

Units:
- CPU: `1` = one core; `500m` = half a core ("m" = millicores, thousandths).
- Memory: `Mi`, `Gi` (mebibytes, gibibytes).

| | requests | limits |
|---|---|---|
| CPU | guaranteed share | **slowed down** (throttled) above it |
| Memory | guaranteed amount | **killed** above it (`OOMKilled` — "out of memory") |

What happens with no requests? The scheduler thinks the Pod needs nothing, packs
too many onto one node, and they fight. **Always set requests.**

Common practice:
- Set **memory request = memory limit** (memory can't be taken back without killing).
- Set a **CPU request**; many teams skip the CPU limit to avoid needless slow-downs.

See real usage (needs metrics-server, below):

```bash
kubectl top pods
kubectl top nodes
```

## Part 3 — Autoscaling

### HorizontalPodAutoscaler (HPA) — more Pods when busy

[`examples/11-health-and-resources/hpa.yaml`](../examples/11-health-and-resources/hpa.yaml):

```yaml
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: healthy }
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 50 }   # 50% of the CPU *request*
```

The HPA needs **metrics-server** to measure CPU. Install it on kind:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl -n kube-system patch deploy metrics-server --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

(The patch is only for kind, whose node certificates aren't signed for this.)

Make some load and watch it scale:

```bash
kubectl apply -f examples/11-health-and-resources/
kubectl run load --image=busybox:1.37 --restart=Never -- sh -c \
  'for i in 1 2 3 4 5 6 7 8; do (while true; do wget -qO- http://healthy >/dev/null; done) & done; wait'
kubectl get hpa healthy -w        # watch replicas climb from 2 towards 6
kubectl delete pod load           # stop the load; it scales back down after ~5 minutes
```

> Percentages are of the **request**. No CPU request = the HPA can't work.

### Other kinds of autoscaling

| Tool | Scales | Based on |
|---|---|---|
| **HPA** | number of Pods | CPU, memory, custom metrics |
| **KEDA** | number of Pods (even to zero) | queue length, events, schedules |
| **VPA** (Vertical Pod Autoscaler) | the requests of each Pod | actual usage over time |
| **Cluster Autoscaler / Karpenter** | number of **nodes** | Pods that can't fit anywhere |

HPA adds Pods → nodes fill up → Pods go `Pending` → the cluster autoscaler adds a node.

## Remember

- Readiness = traffic on/off. Liveness = restart. Startup = "still booting".
- Never check dependencies in liveness.
- Always set requests. Memory over the limit = `OOMKilled`.
- HPA scales Pods using metrics-server; node autoscalers add machines.
