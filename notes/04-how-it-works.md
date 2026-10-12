# 04 · How Kubernetes works inside

You can use Kubernetes without knowing this, but debugging gets *much* easier
when you know who does what.

## The big picture

```
                         CONTROL PLANE (the brain)
   kubectl ──────►  ┌───────────────────────────────────────────┐
   (you)            │  API server   ◄──►  etcd (the database)    │
                    │      ▲                                     │
                    │      ├── scheduler  ("which node?")         │
                    │      └── controller manager ("fix it!")    │
                    └──────┬────────────────────────────────────┘
                           │
          ┌────────────────┼─────────────────┐
          ▼                ▼                 ▼
     WORKER NODE      WORKER NODE       WORKER NODE      (the muscle)
     kubelet          kubelet           kubelet
     container        container         container
     runtime          runtime           runtime
     kube-proxy       kube-proxy        kube-proxy
     [pods]           [pods]            [pods]
```

A **cluster** = a **control plane** (decides) + **worker nodes** (do the work).
A **node** is just a computer — a real server, a cloud VM, or (in kind) a Docker container.

## The control plane — like a company's head office

| Part | Job | Analogy |
|---|---|---|
| **API server** | The **only front door**. kubectl, nodes and every other part talk through it. Checks who you are and whether you're allowed. | Reception desk |
| **etcd** | A small, reliable database storing **every object** (desired state + status). Lose it without a backup = lose the cluster. | The filing cabinet |
| **Scheduler** | Watches for new Pods with no node yet, and picks the best node (enough free CPU/memory, rules respected). | The person assigning desks |
| **Controller manager** | Runs many **controllers** — little robots, each running a control loop: "Deployment wants 3 Pods, there are 2 → make one." | Managers checking targets |

On managed services (EKS, GKE, AKS) the cloud runs the control plane for you.

## Each worker node

| Part | Job |
|---|---|
| **kubelet** | The node's agent. Asks the API server "what should run here?", starts containers, runs health checks, reports back. |
| **Container runtime** | Actually runs containers (usually **containerd**). |
| **kube-proxy** | Sets up networking rules so Services (chapter 07) reach the right Pods. |

## What happens when you run `kubectl apply -f deployment.yaml`

1. **kubectl** sends the YAML to the **API server**.
2. API server checks you're allowed, validates it, saves it in **etcd**. Done — kubectl returns.
3. The **Deployment controller** notices: "a Deployment wants 3 Pods, none exist" → creates a ReplicaSet → which creates 3 Pod objects (no node yet).
4. The **scheduler** notices 3 Pods with no node → picks a node for each → saves that.
5. The **kubelet** on each chosen node notices "a Pod is assigned to me" → tells containerd to pull the image and start it.
6. The kubelet keeps reporting **status** ("Running", "Ready") back to the API server.

Notice: **nobody gives orders directly.** Everyone *watches* the API server
and fixes the differences they're responsible for. That's why it's so robust —
if a part restarts, it just looks again and carries on.

## See it yourself

```bash
kubectl get pods -n kube-system -o wide    # the control plane parts run as Pods here!
kubectl get events --sort-by=.lastTimestamp  # the story of what happened, in order
kubectl describe node learn-worker          # capacity, what's running, conditions
```

## Remember

- Control plane = API server (front door) + etcd (memory) + scheduler (placement) + controllers (fixers).
- Nodes run kubelet (agent) + container runtime + kube-proxy.
- Everything works by **watching** the API server and fixing differences.
