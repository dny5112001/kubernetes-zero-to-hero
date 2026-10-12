# 08 · Ingress and Gateway API — one front door for web traffic

## The problem

You have 10 web apps. A LoadBalancer Service each = 10 cloud load balancers =
10 bills and 10 addresses. You'd rather have **one** entrance that sends
`shop.example.com` to the shop and `example.com/api` to the API.

## Ingress = the receptionist

An **Ingress** is a set of **routing rules** for web (HTTP/HTTPS) traffic:
"this website address / path → that Service".

An **Ingress controller** is the program that actually **does** the routing.
Kubernetes doesn't include one — you install one. Popular choices: **Traefik**,
**HAProxy**, **Contour**, **Kong**, **NGINX Inc's** controller, and cloud ones
like the **AWS Load Balancer Controller**.

> The once-very-popular community **ingress-nginx** project was retired in
> 2026. Pick a different controller for new setups.

```
browser ──► Ingress controller (one entrance) ──► Service blue  ──► Pods
                    reads the Ingress rules     └► Service green ──► Pods
```

## Install a controller (Traefik) on kind

```bash
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik -n traefik --create-namespace \
  -f examples/08-ingress/helm/traefik-values.yaml --wait
```

The values file sets the controller's Service to **NodePort 30080**, which
`kind-cluster.yaml` maps to **http://localhost:8090**.

## Two apps behind one door

```bash
kubectl apply -f examples/08-ingress/
```

[`ingress.yaml`](../examples/08-ingress/ingress.yaml):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: colours
spec:
  ingressClassName: traefik         # which controller should handle this
  rules:
    - host: blue.localhost          # route by website name...
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: blue, port: { number: 80 } }
    - http:                         # ...or by path (any host)
        paths:
          - path: /green
            pathType: Prefix
            backend:
              service: { name: green, port: { number: 80 } }
```

```bash
curl http://blue.localhost:8090/     # Name: blue
curl http://localhost:8090/green     # Name: green
```

(`*.localhost` addresses always point to your own computer.)

`pathType: Prefix` matches `/green`, `/green/anything`. `Exact` matches only `/green`.

## HTTPS

Add a `tls` section pointing to a Secret holding the certificate:

```yaml
spec:
  tls:
    - hosts: [shop.example.com]
      secretName: shop-tls
```

In real clusters, **cert-manager** gets free certificates from Let's Encrypt
and renews them automatically.

## Gateway API — the newer way

**Gateway API** is the modern replacement for Ingress. Same idea, but clearer
and more powerful (header-based routing, traffic splitting for canary
releases, TCP/gRPC), with roles split between teams:

| Object | Who owns it | Says |
|---|---|---|
| `GatewayClass` | platform team | "this is the controller we use" |
| `Gateway` | platform team | "listen on port 443 for *.example.com" |
| `HTTPRoute` | app team | "send /api to my Service" |

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: green
spec:
  parentRefs:
    - name: main-gateway
  rules:
    - matches:
        - path: { type: PathPrefix, value: /green }
      backendRefs:
        - name: green
          port: 80
```

Most controllers (including Traefik) support both. **Learn Ingress first**
(it's everywhere), then Gateway API (it's where things are heading).

## Try it

1. Install Traefik and apply the example. Test both URLs.
2. Add a rule for `green.localhost` → `green`. Apply. Test.
3. Scale `blue` to 0 and curl it — you'll get an error from Traefik (no Pods behind it). Scale back.

## Remember

- Ingress = routing rules for web traffic; an Ingress **controller** does the work (install one).
- One entrance, many apps, by host name and path.
- Gateway API is the newer, more powerful version of the same idea.
