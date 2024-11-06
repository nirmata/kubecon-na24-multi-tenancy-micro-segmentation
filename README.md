# kcna24-multi-tenancy-micro-segmentation

This repository contains a demo of using Kyverno and Cilium to automate multi-tenancy and micro-segmentation for Kubernetes.

The demo is based on the following KubeCon NA 2024 presentation: [Micro-Segmentation and Multi-Tenancy: The Brown M&Ms of Platform Engineering](https://sched.co/1i7q)

Multi-tenancy is enforced based on namespaces and `workspaces`, which provide a segmentation boundary for a collection of namepaces.

The application for the demo is Guestbook. It consists of 2 tiers:
- `Frontend`: the Guestbook PHP app
- `Backend`: a Redis leader and follower

Kyverno policies are used to auto-generate namespace and workspace level Cilium network policies and provide guardrails for application network policies based on application tiers.

# Demo

Clone this repo and cd to the base directory:

```sh
git clone https://github.com/jimbugwadia/kcna24-multi-tenancy-micro-segmentation; cd kcna24-multi-tenancy-micro-segmentation
```

## Setup

### 1. Create kind cluster

```
kind create cluster --config=manifests/kind-config.yaml --name=kcna24mnms
```

### 2. Install Cilium

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

### 2. Install Kyverno

From https://kyverno.io/docs/installation/methods/#install-kyverno-using-helm


```sh
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

Configure Kyverno permissions to allow managing Cilium network policies:

```sh
kubectl apply -f manifests/kyverno-rbac.yaml
```

### 3. Install Kyverno policies

```sh
kubectl apply -f manifests/policies/
```

Kyverno policies will be installed for the following:

- [x] Require a `workspace` and `tier` labels for each namespace
- [x] Automatically add the namespace `workspace` and `tier` labels to pods
- [x] Generate network policy based on `allow-dns-traffic` label on namespace
- [x] Generate network policy based on `allow-ns-traffic` label on namespace
- [x] Generate network policy based on `allow-workspace-traffic` label on namespace
- [x] Restrict images by tier i.e. eg allow `redis` image in the `backend` tier
- [x] Do not allow egress traffic from `backend` tier
- [x] Do not allow arbitrary ingress traffic to `backend` tier


# Demo

## 1. Test Kyverno policies

### Require namespace labels

Try creating a namespace `test`:

```sh
kubectl create ns test
```

This will be blocked by Kyverno:

```sh
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:

resource Namespace//test was blocked due to the following policies

require-labels:
  check-ns: 'validation error: The namespace must have a labels workspace and tier.
    rule check-ns failed at path /metadata/labels/tier/
```

Use the provided sample with required labels:

```sh
kubectl apply -f manifests/ns.yaml
```

This should succeed.

### Mutate pod labels

Run a redis image and check labels:

```sh
k -n test run redis --image redis --dry-run=server -o yaml | grep -e "tier" -e "workspace"
```

### Generate network policies

Check the Cilium network policies:

```sh
k -n test get cnp
```

This should show the default deny-all policy:

```sh
NAME       AGE   VALID
deny-all   10m   True
```

Add a label `allow-dns-traffic`:

```sh
kubectl label ns test allow-dns-traffic=true
```

Check the Cilium network policies again. This should now show an additional `allow-dns` policy:

```sh
k -n test get cnp
NAME        AGE   VALID
allow-dns   5s    True
deny-all    13m   True
```

Try setting the label to `false`:

```sh
kubectl label ns test allow-dns-traffic=false --overwrite
```

Then recheck the policy, the `allow-dns` policy should be deleted.

### Restrict images

Run an unauthorized image in the `backend` tier:

```sh
k -n test run nginx --image nginx --dry-run=server
```

This will be blocked by Kyverno:

```sh
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/test/nginx was blocked due to the following policies

check-images:
  backend-images: 'validation failure: image nginx is not allowed in the backend tier'
```

Run a valid image; this should be allowed:

```sh
kubectl -n test run redis --image=redis --dry-run=server
pod/redis created (server dry run)
```

### Apply namespace labels to pods

Run the following command to make sure the namespace labels are applied to pods:

```sh
kubectl -n test run redis --image redis --dry-run=server -o yaml | grep -e "tier" -e "workspace"
```

### Restrict traffic to the backend tier

Try creating a Cilium network policy that allows internet traffic to the `backend` tier:

```sh
kubectl -n test apply -f manifests/cnp-ingress-world.yaml
```

This will be blocked by kyverno:

```sh
Error from server: error when creating "manifests/cnp-ingress-world.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource CiliumNetworkPolicy/test/backend-ingress was blocked due to the following policies

check-network-policies:
  check-ingress-backend: only ingress traffic from the frontend tier is allowed; fromEntities
    are not allowed
```

You can also try the policy that allows egress traffic from the `backend` tier:

```sh
kubectl -n test apply -f manifests/cnp-egress-world.yaml
```

### Cleanup

```sh
kubectl delete ns test
```

## 2. Run the Guestbook app in 2 different workspaces

The Guestbook app a 2-tier application with `guestbook` running in the `frontend` tier and `redis` running in the `backend` tier. The application is configured across 2 namespaces, one for each tier.

Create 2 instances of the Guestbook application in different workspaces `wks1` and `wks2`. A workspace is a network segmentation boundary used to isolate applications.

```sh
helm install guestbook1 charts/guestbook/ --set workspace=wks1
helm install guestbook2 charts/guestbook/ --set workspace=wks2
```

Check connectity for the applications:

```sh
kubectl port-forward service/guestbook 8080:80 -n wks1-guestbook
```

Navigate to http://localhost:8080 and make sure you an enter a value and it is shown as a guestbook entry after clicking the submit button.

Repeat the test for `guestbook2` in `wks2`.

## 3. Test network connectivity within and across workspaces

Attach `netshoot` as a debug container to the `guestbook` pod in the `frontend` tier:

```sh
kubectl -n wks1-guestbook get pods
NAME                         READY   STATUS    RESTARTS   AGE
guestbook-7dc7786f46-78hkk   1/1     Running   0          3m7s
```

Note: you will need to update the pod number to match yours:

```sh
kubectl -n wks1-guestbook debug -i -t --image nicolaka/netshoot guestbook-7dc7786f46-78hkk
```

In the netshoot session check connectivity from the frontend pod in wks1 to the backend service in wks1:

```sh
nc -w 2 -v redis-leader.wks1-redis.svc.cluster.local 6379
```

This should show a message similar to:

```sh
Connection to redis-leader.wks1-redis.svc.cluster.local (10.11.75.0) 6379 port [tcp/redis] succeeded!
```

Now, try connecting across workspaces:

```sh
nc -w 2 -v redis-leader.wks2-redis.svc.cluster.local 6379
```

This should timeout and fail:

```sh
nc: connect to redis-leader.wks2-redis.svc.cluster.local (10.11.147.254) port 6379 (tcp) timed out: Operation in progress
```

Cleanup:

```sh
helm uninstall guestbook1
helm uninstall guestbook2
```
