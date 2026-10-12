# 13 · Other workload types — and where Pods get placed

## Which workload type do I need?

| Type | Use for | Example |
|---|---|---|
| **Deployment** | apps that run forever, any copy is as good as another | web servers, APIs |
| **StatefulSet** | apps needing stable names + their own disk | databases, Kafka (ch. 10) |
| **DaemonSet** | exactly **one Pod on every node** | log collectors, monitoring agents |
| **Job** | a task that **runs to completion** | database migration, report, backup |
| **CronJob** | a Job **on a schedule** | nightly backup, hourly clean-up |

## Job — run until done

[`examples/13-workload-types/job.yaml`](../examples/13-workload-types/job.yaml):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: count-to-five
spec:
  backoffLimit: 2                  # retry at most 2 times on failure
  ttlSecondsAfterFinished: 600     # tidy up 10 minutes after it finishes
  template:
    spec:
      restartPolicy: Never         # Jobs must use Never or OnFailure
      containers:
        - name: counter
          image: busybox:1.37
          command: ["sh", "-c", "for i in 1 2 3 4 5; do echo $i; sleep 1; done; echo done"]
```

```bash
kubectl apply -f examples/13-workload-types/job.yaml
kubectl wait --for=condition=complete job/count-to-five
kubectl logs job/count-to-five     # 1 2 3 4 5 done
```

Run many in parallel with `completions` and `parallelism`.

## CronJob — a Job on a timer

```yaml
spec:
  schedule: "*/1 * * * *"      # every minute
```

Schedule format (same as Linux cron): `minute hour day-of-month month day-of-week`

| Schedule | Means |
|---|---|
| `*/5 * * * *` | every 5 minutes |
| `0 2 * * *` | every day at 02:00 |
| `0 9 * * 1` | Mondays at 09:00 |

Times are in UTC unless you set `timeZone: "Asia/Kolkata"`.

```bash
kubectl apply -f examples/13-workload-types/cronjob.yaml
kubectl get cronjob,jobs        # a new Job appears each minute
kubectl create job manual-run --from=cronjob/every-minute   # run it now
```

## DaemonSet — one per node

[`examples/13-workload-types/daemonset.yaml`](../examples/13-workload-types/daemonset.yaml):

```bash
kubectl apply -f examples/13-workload-types/daemonset.yaml
kubectl get pods -l app=node-reporter -o wide
# node-reporter-7n8p2   learn-worker2
# node-reporter-xxmzm   learn-worker
```

Only **2** Pods for **3** nodes? The control-plane node has a **taint** (below)
that keeps normal Pods off it. A real log collector would add a matching
**toleration** to run there too.

Add a node → the DaemonSet automatically puts a Pod on it.

## Where do Pods go? Controlling placement

Normally the scheduler decides. You can guide it:

### nodeSelector — "only nodes with this label"

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

```bash
kubectl label node learn-worker disktype=ssd
```

### Affinity — preferences and rules

```yaml
affinity:
  podAntiAffinity:            # "spread my copies across different nodes"
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels: { app: whoami }
          topologyKey: kubernetes.io/hostname
```

Simpler for spreading: `topologySpreadConstraints` (spread evenly across nodes or zones).

### Taints and tolerations — "keep out, unless..."

A **taint** on a node repels Pods. A **toleration** on a Pod lets it ignore that taint.

```bash
kubectl taint node learn-worker2 gpu=true:NoSchedule   # only GPU workloads here
```

```yaml
tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
```

Like a "Staff only" sign on a door (taint), and a staff badge (toleration).
Remove a taint: `kubectl taint node learn-worker2 gpu=true:NoSchedule-` (note the `-`).

### Draining a node for maintenance

```bash
kubectl cordon learn-worker            # no new Pods here
kubectl drain learn-worker --ignore-daemonsets --delete-emptydir-data   # move Pods away
kubectl uncordon learn-worker          # open again
```

A **PodDisruptionBudget** stops a drain from taking down too many copies at once:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: whoami
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: whoami }
```

## Remember

- Deployment (forever, interchangeable), StatefulSet (stable + own disk), DaemonSet (one per node), Job (run to finish), CronJob (on a schedule).
- Guide placement with nodeSelector, affinity, spread constraints; keep Pods off nodes with taints.
- `cordon` / `drain` / `uncordon` for maintenance; PodDisruptionBudgets keep enough copies up.
