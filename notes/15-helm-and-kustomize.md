# 15 · Packaging: Helm and Kustomize

## The problem

A real app is 6–10 YAML files (Deployment, Service, Ingress, ConfigMap,
HPA...). And you need **slightly different** versions for dev and prod:
different replicas, image tags, hostnames. Copy-pasting folders leads to drift
and mistakes. Two popular tools solve this differently.

| | **Kustomize** | **Helm** |
|---|---|---|
| Idea | plain YAML + **patches** on top | YAML **templates** with blanks to fill in |
| Analogy | a photocopy with sticky notes changing a few lines | a form letter: "Dear {{name}}" |
| Built into kubectl? | yes (`kubectl apply -k`) | no, separate `helm` tool |
| Great for | your own apps, per-environment tweaks | installing other people's software; reusable packages |

Many teams use both: **Helm to install third-party tools** (Traefik, cert-manager,
Prometheus), **Kustomize for their own apps**.

## Kustomize

```
examples/15-packaging/kustomize/
├── base/                     # the shared, normal YAML
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml    # lists the files
└── overlays/
    ├── dev/kustomization.yaml    # "the base, but..."
    └── prod/kustomization.yaml
```

`overlays/prod/kustomization.yaml`:

```yaml
namespace: prod          # put everything in the prod namespace
namePrefix: prod-        # web -> prod-web
resources:
  - ../../base
  - namespace.yaml
labels:
  - pairs: { env: prod }
replicas:
  - name: web
    count: 3             # 3 copies in prod
patches:
  - target: { kind: Deployment, name: web }
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: production
```

```bash
kubectl kustomize examples/15-packaging/kustomize/overlays/prod   # preview the final YAML
kubectl apply -k examples/15-packaging/kustomize/overlays/dev
kubectl apply -k examples/15-packaging/kustomize/overlays/prod
kubectl get deploy -n dev     # dev-web    1/1
kubectl get deploy -n prod    # prod-web   3/3
```

Other handy Kustomize features: `images:` (change image tags without patches),
`configMapGenerator` (builds ConfigMaps from files and adds a hash to the name,
so changing config automatically rolls the Pods).

## Helm

Helm is the **app store** of Kubernetes.

- A **chart** = a package of templates + default settings (`values.yaml`).
- A **release** = one installed copy of a chart, with a name.
- A **repository** = where charts are published.

### Installing someone else's software

```bash
helm repo add traefik https://traefik.github.io/charts
helm search repo traefik
helm show values traefik/traefik > defaults.yaml      # see every setting
helm install traefik traefik/traefik -n traefik --create-namespace -f my-values.yaml
helm list -A
helm upgrade traefik traefik/traefik -n traefik -f my-values.yaml
helm uninstall traefik -n traefik
```

Find charts on [Artifact Hub](https://artifacthub.io). Many are now published
as OCI images: `helm install x oci://registry.example.com/charts/x`.

### Your own chart

[`examples/15-packaging/helm/hello-chart`](../examples/15-packaging/helm/hello-chart):

```
hello-chart/
├── Chart.yaml            # name and version
├── values.yaml           # default settings
└── templates/
    ├── deployment.yaml   # YAML with {{ }} blanks
    ├── service.yaml
    └── NOTES.txt         # message shown after install
```

`values.yaml`:
```yaml
replicas: 1
image:
  repository: traefik/whoami
  tag: v1.11
greeting: hello-from-helm
```

`templates/deployment.yaml` (part):
```yaml
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicas }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          env:
            - name: WHOAMI_NAME
              value: {{ .Values.greeting | quote }}
```

`{{ .Values.x }}` = from values; `{{ .Release.Name }}` = the name you install it as.

```bash
cd examples/15-packaging/helm
helm lint hello-chart                                # check for mistakes
helm template hello hello-chart --set replicas=3     # preview the final YAML
helm install hello hello-chart --set replicas=2
helm upgrade hello hello-chart --set replicas=3      # revision 2
helm history hello
helm rollback hello 1                                # back to 2 replicas
helm uninstall hello
```

Helm remembers every revision, so **rollback** is one command.

### Helm tips

- Keep settings in a values file per environment (`values-prod.yaml`) in git, not long `--set` lines.
- Pin chart versions: `--version 41.7.1`.
- `helm upgrade --install` = install if missing, upgrade if present (handy in CI).
- `--atomic` (Helm 3) / `--rollback-on-failure` (Helm 4) undoes a failed upgrade automatically.

## Remember

- Kustomize = base YAML + per-environment patches; built into kubectl (`-k`).
- Helm = templated packages with values; install, upgrade, rollback releases.
- Always preview: `kubectl kustomize` / `helm template`.
