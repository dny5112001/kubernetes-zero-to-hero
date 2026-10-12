# 10 · Storage — disks that survive restarts

## The problem

A container's files disappear when it restarts (chapter 02). Fine for a web
server, a disaster for a database.

## Volumes: three levels

| Kind | Lives as long as | Use for |
|---|---|---|
| `emptyDir` | the **Pod** | scratch space, sharing files between containers in a Pod (chapter 05) |
| `configMap` / `secret` | the object | settings as files (chapter 09) |
| **PersistentVolume** | **independently** of any Pod | real data: databases, uploads |

## Persistent storage in three objects

```
Pod ──uses──► PersistentVolumeClaim ──bound to──► PersistentVolume ──is──► a real disk
              ("I need 10Gi")                    (the actual 10Gi)         (AWS EBS, local disk...)
                                ▲
                       StorageClass ("how to make disks")
```

- **PersistentVolumeClaim (PVC)** — the app's **request**: "I need 100Mi I can write to".
- **PersistentVolume (PV)** — the actual piece of storage.
- **StorageClass** — the **recipe for making PVs automatically** (fast SSD, cheap HDD, which cloud disk type).

Normally you only write the **PVC**. The StorageClass creates the PV for you
("dynamic provisioning"). kind comes with a StorageClass called `standard`:

```bash
kubectl get storageclass
```

Like ordering at a restaurant: you (PVC) say "a large pizza"; the kitchen
(StorageClass) makes one (PV) and brings it to your table.

## Try it: data that survives

[`examples/10-storage/pvc-and-pod.yaml`](../examples/10-storage/pvc-and-pod.yaml)
asks for a 100Mi disk, and a Pod writes the date into it each time it starts.

```bash
kubectl apply -f examples/10-storage/pvc-and-pod.yaml
kubectl get pvc                    # STATUS: Bound
kubectl logs disk-writer           # one line

kubectl delete pod disk-writer
kubectl apply -f examples/10-storage/pvc-and-pod.yaml
kubectl logs disk-writer           # TWO lines — the data survived the Pod
```

### Access modes

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | one **node** can mount it read-write (most cloud disks) |
| `ReadWriteOncePod` | only one **Pod** |
| `ReadOnlyMany` (ROX) | many nodes, read-only |
| `ReadWriteMany` (RWX) | many nodes read-write (needs network storage like AWS EFS, NFS) |

### Reclaim policy

When you delete the PVC, what happens to the data? `Delete` (default for
dynamic disks — **gone**) or `Retain` (kept for manual recovery). For important
data, use a StorageClass with `Retain`, and **take backups** (e.g. **Velero**).

## StatefulSets — for databases and other "pets"

A Deployment treats Pods as **identical and replaceable** (cattle): random
names, shared or no storage. Databases need more:
- **Stable names**: `redis-0`, `redis-1` — always the same.
- **Their own disk each**, which follows them if they move.
- **Ordered start-up**: `-0` first, then `-1`...

That's a **StatefulSet**.

[`examples/10-storage/statefulset.yaml`](../examples/10-storage/statefulset.yaml):
2 Redis Pods, each with its own disk via `volumeClaimTemplates`, plus a
**headless Service** so each one has its own DNS name (`redis-0.redis`).

```bash
kubectl apply -f examples/10-storage/statefulset.yaml
kubectl get pods -l app=redis            # redis-0, redis-1
kubectl get pvc                          # data-redis-0, data-redis-1

kubectl exec redis-0 -- redis-cli set greeting hi
kubectl delete pod redis-0               # it comes back with the SAME name and SAME disk
kubectl exec redis-0 -- redis-cli get greeting    # hi
```

## Should you run databases in Kubernetes?

You *can*, and operators (e.g. **CloudNativePG** for Postgres) make it much
easier. But a **managed database** (AWS RDS, Cloud SQL) handles backups,
failover and upgrades for you. Many teams run apps in Kubernetes and keep the
main database managed. Start there.

## Remember

- Container files are temporary; use PersistentVolumeClaims for real data.
- PVC = request, PV = the disk, StorageClass = how disks are made.
- StatefulSet = stable names + a disk per Pod — for databases and similar.
- Back up anything important.
