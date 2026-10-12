# 14 · Security

Kubernetes is open by default: Pods can talk to all Pods, containers often run
as root, and a stolen login can do a lot. This chapter locks it down in four layers.

```
1. Who can use the API?        → RBAC + service accounts
2. What can a container do?    → securityContext + Pod Security Standards
3. Who can talk to whom?       → NetworkPolicies
4. What's inside the image?    → image scanning, trusted registries
```

## 1. RBAC — who can do what

**RBAC** = Role-Based Access Control. Four objects:

| Object | Says | Scope |
|---|---|---|
| **Role** | "these actions on these things are allowed" | one namespace |
| **ClusterRole** | same | whole cluster |
| **RoleBinding** | "give this Role to this person/app" | one namespace |
| **ClusterRoleBinding** | same | whole cluster |

**Who** can be a **User** (a person, from your login system), a **Group**, or a
**ServiceAccount** (an identity for **apps** running in Pods).

[`examples/14-security/rbac.yaml`](../examples/14-security/rbac.yaml) — an app
identity that may **only read** Pods:

```yaml
kind: Role
metadata:
  name: read-pods
rules:
  - apiGroups: [""]                    # "" = core things (Pods, Services...)
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]   # read-only verbs
```

Check it:

```bash
kubectl apply -f examples/14-security/rbac.yaml
kubectl auth can-i list pods   --as=system:serviceaccount:default:pod-reader   # yes
kubectl auth can-i delete pods --as=system:serviceaccount:default:pod-reader   # no
```

Verbs: `get`, `list`, `watch` (read), `create`, `update`, `patch`, `delete` (write).

Rules:
- **Least privilege**: give the smallest set of verbs and resources needed.
- Avoid `cluster-admin` and `*` wildcards for people and apps.
- Most apps don't need to talk to the Kubernetes API at all → `automountServiceAccountToken: false`.
- On clouds, link service accounts to cloud permissions without keys
  (AWS **EKS Pod Identity** / IRSA, GKE Workload Identity).

## 2. Locking down containers — securityContext

[`examples/14-security/secure-pod.yaml`](../examples/14-security/secure-pod.yaml):

```yaml
spec:
  automountServiceAccountToken: false
  securityContext:                       # for the whole Pod
    runAsNonRoot: true                   # refuse to start as root
    runAsUser: 10001
    seccompProfile: { type: RuntimeDefault }   # block risky system calls
  containers:
    - name: app
      securityContext:                   # for this container
        allowPrivilegeEscalation: false  # can't become root later
        readOnlyRootFilesystem: true     # can't change its own files
        capabilities: { drop: ["ALL"] }  # no special Linux powers
      volumeMounts:
        - { name: tmp, mountPath: /tmp } # a writable scratch folder, if the app needs one
```

```bash
kubectl apply -f examples/14-security/secure-pod.yaml
kubectl logs locked-down
# uid=10001 ...
# can write /tmp
# touch: /oops: Read-only file system
# cannot write /
```

If someone breaks into this container, there's very little they can do.

### Pod Security Standards — enforce it for a whole namespace

Kubernetes has three built-in levels: `privileged` (anything goes), `baseline`
(blocks the worst things), `restricted` (the secure-pod settings above).
Turn them on with a namespace label:

```bash
kubectl label namespace team-a pod-security.kubernetes.io/enforce=restricted
```

Now Pods that don't meet "restricted" are rejected in `team-a`. (Use `warn=`
instead of `enforce=` to try it first.) For custom rules, teams add policy tools
like **Kyverno** or **OPA Gatekeeper**, or built-in **ValidatingAdmissionPolicy**.

## 3. NetworkPolicies — firewalls between Pods

By default, **every Pod can reach every other Pod**. A NetworkPolicy changes
that: once a Pod is selected by any policy, **only allowed traffic gets in**.

[`examples/14-security/network-policy.yaml`](../examples/14-security/network-policy.yaml):

```yaml
spec:
  podSelector:
    matchLabels: { app: whoami }       # protect these Pods
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels: { access: allowed }   # only from Pods with this label
      ports:
        - port: 80
```

```bash
kubectl apply -f examples/06-deployments/ -f examples/07-services/ -f examples/14-security/network-policy.yaml

kubectl run stranger --rm -i --restart=Never --image=busybox:1.37 -- wget -qO- -T 4 http://whoami
# wget: download timed out                      <- blocked

kubectl run friend --rm -i --restart=Never --labels=access=allowed --image=busybox:1.37 -- wget -qO- -T 4 http://whoami
# Hostname: whoami-...                           <- allowed
```

> ⚠️ This policy also blocks traffic coming from **outside** — the NodePort
> (`curl localhost:8091`) and the ingress controller now time out too, because
> they aren't Pods with `access: allowed`. A forgotten NetworkPolicy is a classic
> "why can't I reach my app?" cause. When you're done:
> `kubectl delete -f examples/14-security/network-policy.yaml`

Good practice: a **"deny all" policy per namespace**, then allow only what's
needed. Remember to allow DNS (port 53) for outgoing (`Egress`) policies.

> NetworkPolicies need a network plugin that supports them. kind's default one
> does; on clouds, Calico or Cilium are common.

## 4. Images and the supply chain

- Use **small, trusted base images** (distroless, Alpine, Chainguard, slim).
- **Scan images** for known vulnerabilities in CI (Trivy, Grype).
- Pin images by version (or by digest `@sha256:...`), never `latest`.
- Only allow images from your own registry (a policy tool can enforce this).
- Keep Kubernetes itself up to date.

## Remember

- RBAC: least privilege; check with `kubectl auth can-i`.
- Run as non-root, read-only filesystem, drop all capabilities; enforce with Pod Security Standards.
- Pods can all talk to each other until you add NetworkPolicies.
- Scan and pin your images.
