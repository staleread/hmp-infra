# hmp-infra

Infrastructure configuration for [HearMyPaper Backend](https://github.com/staleread/hmp-server).

## Prerequisites

```bash
sudo pacman -S minikube kubectl helm argocd
```

## Cluster setup

```bash
minikube start
minikube addons enable ingress
```

## Apply secrets and config

These are applied once manually and are not managed by ArgoCD (they contain credentials):

```bash
kubectl apply -f k8s/config.yaml
kubectl apply -f k8s/secret.yaml
```

## Install the app chart (without ArgoCD)

Pull chart dependencies:

```bash
helm dependency update helm/hearmypaper/
```

Install:

```bash
helm install hearmypaper helm/hearmypaper/
```

Get the Ingress IP and open `http://<IP>/docs`:

```bash
kubectl describe ingress hearmypaper | grep Address
```

### Upgrade and rollback

```bash
helm upgrade hearmypaper helm/hearmypaper/ --set replicaCount=2
helm rollback hearmypaper
helm history hearmypaper
```

### Cleanup

```bash
helm uninstall hearmypaper
kubectl delete -f k8s/config.yaml
kubectl delete -f k8s/secret.yaml
```

---

## ArgoCD setup

### Install ArgoCD

```bash
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace argocd
```

Wait for all pods to reach Running:

```bash
kubectl get pods -n argocd --watch
```

### Access the UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `http://localhost:8080` (accept the self-signed certificate).

### Initial admin password

The password is auto-generated during install and stored in a Secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Login with username `admin` and the password above. After logging in and setting a new password, delete the initial secret:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

### Create an ArgoCD Application

Log in via the CLI first:

```bash
argocd login localhost:8080
```

Create the Application pointed at this repo:

```bash
argocd app create hearmypaper \
  --repo https://github.com/staleread/hmp-infra.git \
  --path helm/hearmypaper \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

Check status:

```bash
argocd app get hearmypaper
```

Expected: `Health Status: Healthy`, `Sync Status: Synced`.

### GitOps workflow

Any change pushed to `helm/hearmypaper/` in this repo will be picked up by ArgoCD within ~3 minutes and automatically applied to the cluster. To trigger sync immediately:

```bash
argocd app sync hearmypaper
```
