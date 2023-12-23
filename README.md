# Deploy Gloo Gateway v2 with Argo CD

## Introduction
GitOps is becoming increasingly popular approach to manage Kubernetes components. It works by using Git as a single source of truth for declarative infrastructure and applications, allowing your application definitions, configurations, and environments to be declarative and version controlled. This helps to make these workflows automated, auditable, and easy to understand.

## Purpose of this Tutorial
The main goal of this tutorial is to showcase how Gloo Gateway V2 components can seamlessly integrate into a GitOps workflow, with Argo CD being our tool of choice. We'll guide you through the installation of Argo CD and Gloo Gateway V2 and afterwards we will walk through some API Gateway use cases.

## High Level Architecture
![High Level Architecture](.images/single-cluster-arch1.png)

## Prerequisites
This tutorial assumes a single Kubernetes cluster for demonstration. Instructions have been validated on k3d, as well as in EKS and GKE. Please note that the setup and installation of Kubernetes are beyond the scope of this guide. Ensure that your cluster contexts are named `gloo` by running:
```
% kubectl config get-contexts
CURRENT   NAME   CLUSTER    AUTHINFO         NAMESPACE
*         gloo   k3d-gloo   admin@k3d-gloo   
```

#### Renaming Cluster Context
If your local clusters have a different context name, you will want to have it match the expected context name(s)
```
kubectl config rename-context <k3d-your_cluster_name> gloo
```

### Installing Argo CD	
Let's start by deploying Argo CD to our `gloo` cluster context

Create Argo CD namespace
```
kubectl create namespace argocd --context gloo
```

Deploy Argo CD 2.8.0 using the [non-HA YAML manifests](https://github.com/argoproj/argo-cd/releases)
```
until kubectl apply -k https://github.com/solo-io/gitops-library.git/argocd/deploy/insecure-rootpath/ --context gloo > /dev/null 2>&1; do sleep 2; done
```

Check to see Argo CD status
```
kubectl get pods -n argocd --context gloo
```

Output should look similar to below
```
% kubectl get pods -n argocd --context gloo
NAME                                  READY   STATUS    RESTARTS   AGE
argocd-redis-74d8c6db65-lj5qz         1/1     Running   0          5m48s
argocd-dex-server-5896d988bb-ksk5j    1/1     Running   0          5m48s
argocd-application-controller-0       1/1     Running   0          5m48s
argocd-repo-server-6fd99dbbb5-xr8ld   1/1     Running   0          5m48s
argocd-server-7dd7894bd7-t92rr        1/1     Running   0          5m48s
```

We can also change the password to: `admin / solo.io`:
```
# bcrypt(password)=$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy
# password: solo.io
kubectl --context gloo -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```

#### Navigating to Argo CD UI
At this point, we should be able to access our Argo CD server using port-forward at http://localhost:9999
```
kubectl port-forward svc/argocd-server -n argocd 9999:443 --context gloo
```

## Installing Gloo Gateway V2
Gloo Gateway V2 can be installed and configured easily using Helm + Argo CD

Install Gateway API CRDs
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

Next we can deploy Gloo Gateway V2
```
kubectl apply --context gloo -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gloo-gateway-v2
  namespace: argocd
spec:
  destination:
    namespace: gloo-system
    server: https://kubernetes.default.svc
  project: default
  sources:
  - path: gloo-edge/gloo-gateway-v2/2.0.0-beta1
    repoURL: https://github.com/solo-io/gitops-library
    targetRevision: HEAD
    helm:
      valueFiles:
      - values.yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

## Configure Gateway Components
```
kubectl apply --context gloo -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gateway-config
  namespace: argocd
spec:
  destination:
    namespace: gloo-system
    server: https://kubernetes.default.svc
  project: default
  sources:
  - path: environments/gloo-edge/ggv2/gloo-gateway-v2-config/localhost
    repoURL: https://github.com/solo-io/aoa-catalog
    targetRevision: ggv2
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

## Deploy httpbin
```
kubectl apply --context gloo -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: httpbin
  namespace: argocd
spec:
  destination:
    namespace: httpbin
    server: https://kubernetes.default.svc
  project: default
  sources:
  - path: environments/gloo-edge/ggv2/httpbin-app/localhost
    repoURL: https://github.com/solo-io/aoa-catalog
    targetRevision: ggv2
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

