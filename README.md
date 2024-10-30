# kcna24-multi-tenancy-micro-segmentation

# Setup

## 1. Create kind cluster

```
kind create cluster --config=manifests/kind/config.yaml
```

## 2. Install Cilium

From: https://docs.cilium.io/en/latest/installation/kind/#install-cilium

NOTE: The Cilium Helm chart is already downloaded to the [manifests/cilium](./manifests/cilum) folder.
<!-- ``` sh
curl -LO https://github.com/cilium/cilium/archive/main.tar.gz
tar xzf main.tar.gz
cd cilium-main/install/kubernetes
``` -->

``` sh
docker pull quay.io/cilium/cilium:v1.17.0-pre.1
kind load docker-image quay.io/cilium/cilium:v1.17.0-pre.1
```

``` sh
helm install cilium ./charts/cilium \
   --namespace kube-system \
   --set image.pullPolicy=IfNotPresent \
   --set ipam.mode=kubernetes
```

```sh
brew install cilium
```

```sh
cilium status --wait
```

This should show the output similar to:

```sh
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium             Desired: 4, Ready: 4/4, Available: 4/4
DaemonSet              cilium-envoy       Desired: 4, Ready: 4/4, Available: 4/4
Deployment             cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium             Running: 4
                       cilium-envoy       Running: 4
                       cilium-operator    Running: 2
Cluster Pods:          3/3 managed by Cilium
Helm chart version:    1.17.0-dev
Image versions         cilium             quay.io/cilium/cilium-ci:latest: 4
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.30.6-1728346971-e2dfcc576d5152c967479115e0e0a3905be766bb@sha256:8ce0d0514a70a4d9141d946491c9bfe5fd479c1992ab6ef06f9af99ab938d1d9: 4
                       cilium-operator    quay.io/cilium/operator-generic-ci:latest: 2
```

## 2. Install Kyverno

From https://kyverno.io/docs/installation/methods/#install-kyverno-using-helm


```sh
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```


## 2. Run the Guestbook Sample App

```sh
helm install guestbook charts/guestbook/ --set namespacePrefix=guestbook
```

```sh
kubectl port-forward service/frontend 8080:80 -n guestbook-frontend
```

## 3. Check network connectivity


# Kyverno Policies

1. Require a `workspace` label for each namespace
1. Automatically add `workspace` label to pods
1. Generate netpol based on `DNS` label on namespace
1. Generate netpol based on `allow-traffic-within-namespace` label on namespace
1. Only allow `redis` image in the `backend` tier
1. Only allow `frontend` image in the `frontend` tier

Within a workspace:
1. Only allow `frontend` traffic to the `backend` tier
1. Only allow internet traffic to the `frontend` tier