# 19 · Glossary + interview questions

## Glossary (plain English)

| Word | Meaning |
|---|---|
| **Container** | An app packed with everything it needs, running in isolation |
| **Image** | The frozen package a container is started from |
| **Registry** | An online shelf of images (Docker Hub, ECR, GHCR) |
| **Cluster** | A group of computers run together by Kubernetes |
| **Node** | One computer in the cluster |
| **Control plane** | The "brain": API server, etcd, scheduler, controllers |
| **API server** | The single front door every request goes through |
| **etcd** | The database holding the whole cluster's state |
| **Scheduler** | Picks which node each new Pod goes to |
| **Controller** | A loop that keeps fixing reality to match what you asked for |
| **kubelet** | The agent on each node that runs Pods |
| **kubectl** | The command-line remote control |
| **Manifest** | A YAML file describing objects |
| **Desired state** | What you asked for (`spec`); `status` is what's real |
| **Pod** | One or a few containers that live together; the smallest unit |
| **ReplicaSet** | Keeps N identical Pods running |
| **Deployment** | Manages ReplicaSets: updates, rollbacks, scaling |
| **StatefulSet** | Pods with stable names and their own disks |
| **DaemonSet** | One Pod on every node |
| **Job / CronJob** | A task that runs to completion / on a schedule |
| **Service** | A stable name and address for a group of Pods |
| **Endpoints / EndpointSlice** | The list of Pod addresses behind a Service |
| **Ingress** | Rules for routing web traffic into Services |
| **Ingress controller** | The program that carries out Ingress rules |
| **Gateway API** | The newer, richer replacement for Ingress |
| **ConfigMap / Secret** | Settings / sensitive settings, kept outside images |
| **Volume** | A folder attached to a Pod |
| **PV / PVC** | A piece of storage / a request for storage |
| **StorageClass** | The recipe for creating storage automatically |
| **Namespace** | A named area of the cluster for grouping and limits |
| **Label / Selector** | A key=value tag / a filter that finds things by tag |
| **Annotation** | A key=value note for tools (not used for selecting) |
| **Probe** | A health check (startup, readiness, liveness) |
| **Requests / Limits** | Guaranteed resources / maximum allowed resources |
| **HPA** | Automatically changes the number of Pods based on load |
| **Taint / Toleration** | "Keep out" sign on a node / permission to ignore it |
| **RBAC** | Rules for who can do what |
| **ServiceAccount** | An identity for apps running in Pods |
| **NetworkPolicy** | Firewall rules between Pods |
| **Helm chart** | A package of templated Kubernetes YAML |
| **Kustomize** | Base YAML plus per-environment patches |
| **Operator** | A custom controller that runs a complex app (e.g. a database) for you |
| **CRD** | Custom Resource Definition — teaches Kubernetes a new object type |
| **GitOps** | Git is the source of truth; a tool keeps the cluster matching it |

## Interview questions (with short answers)

**1. What is Kubernetes?**
A system that runs and manages containers across many machines: scheduling,
self-healing, scaling, rolling updates, service discovery and configuration.

**2. Explain the control plane components.**
API server (front door), etcd (state store), scheduler (places Pods),
controller manager (control loops). Nodes run kubelet, a container runtime and kube-proxy.

**3. Pod vs container?**
A Pod wraps one or more containers that share network and storage and are scheduled together.

**4. Why use a Deployment instead of a Pod?**
A plain Pod isn't recreated if it dies. A Deployment keeps the right number of
Pods running and handles rolling updates and rollbacks.

**5. Deployment vs StatefulSet?**
Deployment Pods are interchangeable. StatefulSet Pods have stable names, their
own persistent volumes, and ordered start/stop — for databases and similar.

**6. Explain the Service types.**
ClusterIP (internal), NodePort (a port on every node), LoadBalancer (cloud
load balancer), ExternalName (DNS alias). Headless (`clusterIP: None`) returns Pod IPs.

**7. How does a Service find its Pods?**
By label selector; only Ready Pods are added to its endpoints.

**8. Ingress vs LoadBalancer Service?**
A LoadBalancer exposes one Service. Ingress gives one entry point that routes
HTTP traffic to many Services by host/path, via an ingress controller.

**9. Liveness vs readiness vs startup probes?**
Readiness controls traffic; liveness restarts stuck containers; startup gives
slow apps time to boot before the others start.

**10. Requests vs limits? What's OOMKilled?**
Requests are guaranteed and used for scheduling; limits are maximums. Exceeding
the memory limit kills the container (OOMKilled); exceeding CPU limit throttles it.

**11. How does a rolling update work? How do you roll back?**
New Pods start and become Ready before old ones are removed, controlled by
maxSurge/maxUnavailable. Roll back with `kubectl rollout undo`.

**12. ConfigMap vs Secret? Are Secrets secure?**
Both inject configuration. Secrets are for sensitive data, but only
base64-encoded; security comes from RBAC, encryption at rest and external secret managers.

**13. How do you persist data?**
PersistentVolumeClaims backed by StorageClasses; StatefulSets use volumeClaimTemplates for a disk per Pod.

**14. A Pod is Pending. What do you check?**
`kubectl describe` Events: insufficient resources, node selectors/affinity,
taints, unbound PVCs, quotas.

**15. A Pod is in CrashLoopBackOff. What do you do?**
`kubectl logs --previous`, check command/args, config and Secrets, probes, and memory limits.

**16. What is a namespace? Does it isolate security?**
A logical grouping for names, quotas and RBAC. Not network isolation by itself — that needs NetworkPolicies.

**17. What are taints and tolerations?**
Taints keep Pods off a node unless the Pod has a matching toleration. Used for dedicated or special nodes.

**18. How does the HPA work?**
It reads metrics (via metrics-server) and changes a workload's replica count to
hit a target, e.g. 50% of CPU requests.

**19. What is RBAC?**
Roles define allowed verbs on resources; bindings give roles to users, groups or service accounts.

**20. Helm vs Kustomize?**
Helm templates packages with values and tracks releases with rollback.
Kustomize patches plain YAML per environment and is built into kubectl.

**21. What is GitOps?**
Desired state lives in git; an in-cluster tool (Argo CD, Flux) continuously applies it and corrects drift.

**22. What happens when you run `kubectl apply -f deployment.yaml`?**
API server validates and stores it in etcd → Deployment controller creates a
ReplicaSet → which creates Pods → scheduler assigns nodes → kubelets start containers and report status.

## Where to go next

1. Build the [k8s-terraform-lab](https://github.com/dny5112001/k8s-terraform-lab) and do its exercises.
2. Official tutorials: [kubernetes.io/docs/tutorials](https://kubernetes.io/docs/tutorials/).
3. Practise troubleshooting on purpose: break things in your kind cluster and fix them.
4. Optional certifications: **KCNA** (beginner), **CKAD** (developer), **CKA** (administrator) — CKA/CKAD are hands-on exams in a real terminal.
