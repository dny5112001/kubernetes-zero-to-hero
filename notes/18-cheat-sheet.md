# 18 · Cheat sheet

## Cluster and context

```bash
kind create cluster --config examples/kind-cluster.yaml
kind delete cluster --name learn
kubectl config get-contexts
kubectl config use-context kind-learn
kubectl config set-context --current --namespace=team-a
kubectl cluster-info
kubectl get nodes -o wide
```

## Create, update, delete

```bash
kubectl apply -f file.yaml          # create/update (declarative) — use this
kubectl apply -f folder/
kubectl apply -k overlays/prod      # Kustomize
kubectl diff -f file.yaml           # what would change?
kubectl delete -f file.yaml
kubectl delete pod NAME --now
```

## Look at things

```bash
kubectl get pods [-A] [-n NS] [-o wide] [-w] [--show-labels] [-l app=x]
kubectl get all
kubectl get pod NAME -o yaml
kubectl get deploy NAME -o jsonpath='{.spec.replicas}'
kubectl describe pod NAME
kubectl get events --sort-by=.lastTimestamp
kubectl explain deployment.spec.strategy
kubectl api-resources
kubectl top pods / nodes            # needs metrics-server
```

## Logs and inside

```bash
kubectl logs NAME [-c CONTAINER] [-f] [--previous] [--tail=50]
kubectl logs -l app=x
kubectl exec -it NAME -- sh
kubectl debug -it NAME --image=busybox:1.37 --target=CONTAINER
kubectl run tmp --rm -it --image=busybox:1.37 -- sh
kubectl port-forward svc/NAME 8095:80
kubectl cp NAME:/path/file ./file
```

## Deployments

```bash
kubectl create deployment web --image=nginx:1.29-alpine --replicas=2
kubectl scale deploy/web --replicas=5
kubectl set image deploy/web nginx=nginx:1.29
kubectl rollout status deploy/web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web [--to-revision=N]
kubectl rollout restart deploy/web
kubectl expose deploy/web --port=80          # quick Service
```

## Config

```bash
kubectl create configmap NAME --from-literal=k=v --from-file=file.conf
kubectl create secret generic NAME --from-literal=password=xyz
kubectl get secret NAME -o jsonpath='{.data.password}' | base64 -d
```

## Generate YAML instead of writing it from scratch

```bash
kubectl create deployment web --image=nginx:1.29-alpine --dry-run=client -o yaml > deploy.yaml
kubectl create service clusterip web --tcp=80:80 --dry-run=client -o yaml
kubectl create job once --image=busybox:1.37 --dry-run=client -o yaml -- echo hi
```

## Nodes

```bash
kubectl cordon NODE / uncordon NODE
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data
kubectl label node NODE disktype=ssd
kubectl taint node NODE key=value:NoSchedule[-]
```

## Permissions

```bash
kubectl auth can-i delete pods
kubectl auth can-i list secrets --as=system:serviceaccount:NS:SA
```

## Helm

```bash
helm repo add NAME URL && helm repo update
helm search repo NAME
helm show values REPO/CHART
helm install REL REPO/CHART -n NS --create-namespace -f values.yaml
helm upgrade --install REL REPO/CHART -f values.yaml
helm list -A
helm history REL
helm rollback REL N
helm template REL ./chart     # preview
helm uninstall REL
```

## Status → cause

| Status | Look at |
|---|---|
| Pending | `describe` Events (resources, node selector, taints, PVC) |
| ImagePullBackOff | image name/tag, registry login |
| CrashLoopBackOff | `logs --previous` |
| OOMKilled | memory limit |
| 0/1 Ready | readiness probe |
| Service not working | `get endpointslices` — empty = selector/labels mismatch |

## Object skeletons

```yaml
apiVersion: apps/v1          # Deployment, StatefulSet, DaemonSet
apiVersion: v1               # Pod, Service, ConfigMap, Secret, Namespace, PVC
apiVersion: batch/v1         # Job, CronJob
apiVersion: networking.k8s.io/v1      # Ingress, NetworkPolicy
apiVersion: autoscaling/v2            # HorizontalPodAutoscaler
apiVersion: rbac.authorization.k8s.io/v1   # Role, RoleBinding, ClusterRole...
apiVersion: policy/v1                 # PodDisruptionBudget
```
