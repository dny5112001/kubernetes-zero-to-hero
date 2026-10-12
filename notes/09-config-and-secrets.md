# 09 · ConfigMaps and Secrets — settings outside the image

## Why keep settings outside the image?

You want **one image** that runs in dev, staging and prod. What changes
between them — database address, feature switches, passwords — should be
**plugged in from outside**, not baked in.

- **ConfigMap** = normal settings.
- **Secret** = sensitive settings (passwords, API keys, certificates).

## ConfigMap

[`examples/09-config/configmap.yaml`](../examples/09-config/configmap.yaml):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  APP_COLOUR: "purple"
  APP_MODE: "learning"
  welcome.txt: |
    Hello! This whole file came from a ConfigMap.
```

Quick ways to make one without a file:

```bash
kubectl create configmap app-settings --from-literal=APP_MODE=learning
kubectl create configmap nginx-conf --from-file=nginx.conf
```

## Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:                      # write plain text here...
  DB_PASSWORD: "not-a-real-password"
```

Kubernetes stores it **base64-encoded**. ⚠️ **base64 is not encryption** —
it's just a different way of writing the same text:

```bash
echo bm90LWEtcmVhbC1wYXNzd29yZA== | base64 -d     # not-a-real-password
```

What actually makes Secrets safer than ConfigMaps:
- They can be **access-controlled separately** (RBAC, chapter 14).
- They can be **encrypted in etcd** (a cluster setting; managed clouds usually do this).
- They're kept in memory on nodes, not written to disk.

**Never commit real Secrets to git.** Real setups use:
- **External Secrets Operator** — copies secrets from AWS Secrets Manager / Vault / etc. into the cluster,
- **Sealed Secrets** or **SOPS** — encrypted secrets that are safe to commit,
- the **Secrets Store CSI driver** — mounts secrets straight from a cloud vault.

## Using them in a Pod — two ways

[`examples/09-config/pod-using-config.yaml`](../examples/09-config/pod-using-config.yaml):

### 1. As environment variables

```yaml
envFrom:
  - configMapRef:
      name: app-settings          # every key → an env var
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: DB_PASSWORD          # just one key
```

### 2. As files

```yaml
volumeMounts:
  - name: config-files
    mountPath: /config            # every key → a file in /config
volumes:
  - name: config-files
    configMap:
      name: app-settings
```

```bash
kubectl apply -f examples/09-config/
kubectl logs config-demo
# colour=purple mode=learning password=not-a-real-password
# Hello! This whole file came from a ConfigMap.
```

## Changing settings later

| How it's used | After you change the ConfigMap/Secret |
|---|---|
| Environment variables | **Not** updated. Env vars are read only when the container starts → `kubectl rollout restart deploy/<name>` |
| Files (volume) | Updated inside the Pod after up to about a minute — but the app has to re-read the file |

A common trick (used in the [k8s-terraform-lab](https://github.com/dny5112001/k8s-terraform-lab)):
put a **hash of the config** in a Pod-template annotation, so any config
change automatically rolls the Pods. Helm and Kustomize can do this for you.

## Try it

1. Apply the example and read the logs.
2. Change `APP_COLOUR` to `orange`, apply, delete the Pod and apply it again — logs now say orange.
3. Decode the stored Secret: `kubectl get secret app-secrets -o jsonpath='{.data.DB_PASSWORD}' | base64 -d`. Now you see why base64 isn't protection.

## Remember

- ConfigMap = settings, Secret = sensitive settings. Keep both out of the image.
- Use them as env vars or as files.
- base64 ≠ encryption. Don't commit real Secrets; use an external secret store.
