---
title: Deploy Kuma on Kubernetes
---

To start learning how Kuma works, you run and secure a simple demo application that consists of two services:

- `demo-app`: a web application that lets you increment a numeric counter. It listens on port 5000
- `redis`: data store for the counter

## Prerequisites

- [Helm](https://helm.sh/) - a package manager for Kubernetes
- [minikube](https://minikube.sigs.k8s.io/docs/) - a tool for running local Kubernetes clusters

## Start Kubernetes cluster

Start a new Kubernetes cluster on your local machine by executing the command below. The -p option creates a new profile named 'mesh-zone'."

```sh
minikube start -p mesh-zone
```

You can skip this step if you already have a Kubernetes cluster running.
It can be a cluster running locally or in a public cloud like AWS EKS, GCP GKE, etc.

## Install Kuma

Install Kuma control plane with Helm by executing:

```sh
helm repo add kuma https://kumahq.github.io/charts
helm repo update
helm install --create-namespace --namespace kuma-system kuma kuma/kuma
```

## Deploy demo application

1. Deploy the application

```sh
kubectl apply -f https://raw.githubusercontent.com/kumahq/kuma-counter-demo/master/demo.yaml
kubectl wait -n kuma-demo --for=condition=ready pod --selector=app=demo-app --timeout=90s
```

2. Port-forward the service to the namespace on port 5000:

```sh
kubectl port-forward svc/demo-app -n kuma-demo 5000:5000
```

3. In a browser, go to [127.0.0.1:5000](http://127.0.0.1:5000) and increment the counter.

The demo app includes the `kuma.io/sidecar-injection` label enabled on the `kuma-demo` namespace.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kuma-demo
  labels:
    kuma.io/sidecar-injection: enabled
```

This means that Kuma already knows that it needs to automatically inject a sidecar proxy to every Kubernetes pod in the `kuma-demo` namespace.

## Explore the GUI

You can view the sidecar proxies that are connected to the Kuma control plane.

Kuma ships with a __read-only__ GUI that you can use to retrieve Kuma resources. By default, the GUI listens on the API port which defaults to `5681`.

To access Kuma we need to first port-forward the API service with:

```sh
kubectl port-forward svc/kuma-control-plane -n kuma-system 5681:5681
```

And then navigate to [127.0.0.1:5681/gui](http://127.0.0.1:5681/gui) to see the GUI.


## Introduce zero-trust security

By default, the network is __insecure and not encrypted__. We can change this with Kuma by enabling
the Mutual TLS policy to provision a Certificate Authority (CA) that
will automatically assign TLS certificates to our services (more specifically to the injected data plane proxies running
alongside the services).

We can enable Mutual TLS with a `builtin` CA backend by executing:

```sh
echo "apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  meshServices:
    mode: Exclusive
  mtls:
    enabledBackend: ca-1
    backends:
    - name: ca-1
      type: builtin" | kubectl apply -f -
```

The traffic is now __encrypted and secure__. Kuma does not define default traffic permissions, which
means that no traffic will flow with mTLS enabled until we define a proper MeshTrafficPermission policy.

For now, the demo application won't work.
You can verify this by clicking the increment button again and seeing the error message in the browser.
We can allow the traffic from the `demo-app` to `redis` by applying the following `MeshTrafficPermission`:

```sh
echo "apiVersion: kuma.io/v1alpha1
kind: MeshTrafficPermission
metadata:
  namespace: kuma-demo
  name: redis
spec:
  targetRef:
    kind: MeshSubset
    tags:
      app: redis
  from:
    - targetRef:
        kind: MeshSubset
        tags:
          kuma.io/service: demo-app_kuma-demo_svc_5000
      default:
        action: Allow" | kubectl apply -f -
```

You can click the increment button, the application should function once again.
However, the traffic to `redis` from any other service than `demo-app` is not allowed.

## Cleanup 

```sh
minikube delete -p mesh-zone
```