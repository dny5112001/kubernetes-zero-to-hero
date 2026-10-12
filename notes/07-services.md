# 07 · Services — stable addresses for moving Pods

## The problem

Pods come and go. Each new Pod gets a **new IP address**. If your frontend
talks to your backend by Pod IP, it breaks every time a Pod is replaced. And
with 3 copies, which one should it call?

## The answer: a Service

A **Service** is a **stable name and address** in front of a group of Pods.
It also **spreads traffic** across them (load balancing).

Like a **company phone number**: the people answering change all the time,
but you always dial the same number and someone picks up.

[`examples/07-services/service-clusterip.yaml`](../examples/07-services/service-clusterip.yaml):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  type: ClusterIP
  selector:
    app: whoami        # send traffic to Pods with this label
  ports:
    - port: 80         # the Service's port
      targetPort: 80   # the container's port
```

The `selector` finds Pods by **label**. Any Pod with `app: whoami` that is
**Ready** gets traffic — new ones are added and dead ones removed automatically.

```bash
kubectl apply -f examples/06-deployments/ -f examples/07-services/
kubectl get svc
kubectl get endpointslices -l kubernetes.io/service-name=whoami   # the Pod IPs behind it
```

## Built-in DNS: call apps by name

Every Service gets a DNS name. From any Pod in the same namespace:

```
http://whoami
```

From another namespace: `http://whoami.<namespace>` (full form:
`whoami.default.svc.cluster.local`).

```bash
kubectl run tmp --rm -i --restart=Never --image=busybox:1.37 -- wget -qO- http://whoami
# Hostname: whoami-8d976fdbc-szzsq      <- run it again: a different Pod answers
```

## The Service types

| Type | Reachable from | Use for |
|---|---|---|
| **ClusterIP** (default) | Inside the cluster only | Talking between your own apps (most Services) |
| **NodePort** | A port (30000–32767) on **every node** | Testing; behind your own load balancer |
| **LoadBalancer** | The internet, via a cloud load balancer | Exposing one app on AWS/GCP/Azure |
| **ExternalName** | — | A DNS alias to something outside (`db.example.com`) |

### NodePort, tried for real

[`examples/07-services/service-nodeport.yaml`](../examples/07-services/service-nodeport.yaml)
opens node port `30007`. Our kind config maps that to your laptop:

```bash
curl http://localhost:8091
```

### LoadBalancer

On a cloud, `type: LoadBalancer` asks the cloud for a real load balancer with a
public address (`EXTERNAL-IP`). On kind there's no cloud, so it stays
`<pending>` forever — that's normal. (Tools like `cloud-provider-kind` or
MetalLB can fake it locally.)

One LoadBalancer per app gets expensive. For websites, use **one** front door
for many apps instead: **Ingress** (next chapter).

## Headless Services

`clusterIP: None` = no single address; DNS returns **each Pod's address**
instead. Used with StatefulSets so you can reach a specific Pod like
`redis-0.redis` (chapter 10).

## Quick testing without a Service

```bash
kubectl port-forward svc/whoami 8095:80     # http://localhost:8095
```

## Try it

1. Apply the Deployment and both Services.
2. Run the `wget` test Pod several times — watch different Pods answer.
3. Scale the Deployment to 1 and look at the endpoint slices again.
4. Change the Service's selector to `app: nobody` and `curl localhost:8091` — it
   fails because no Pods match. **A wrong selector is the #1 Service bug.** Change it back.

## Remember

- Service = stable name + load balancing for a group of Pods, found by **labels**.
- Inside the cluster, call apps by name: `http://whoami`.
- ClusterIP for internal, NodePort for testing, LoadBalancer/Ingress for the outside world.
