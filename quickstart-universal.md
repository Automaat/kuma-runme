---
title: Deploy Kuma on Universal
---

This demo shows how to run Kuma in Universal mode on a single machine.

To start learning how Kuma works, you will run and secure a simple demo application that consists of two services:

- `demo-app`: a web application that lets you increment a numeric counter. It listens on port 5000
- `redis`: data store for the counter

## Prerequisites

* [Redis installed, not running](https://redis.io/docs/getting-started/)

## Install Kuma

To download Kuma we will use official installer, it will automatically detect the operating system (Amazon Linux, CentOS, RedHat, Debian, Ubuntu, and macOS) and download Kuma:

```sh
curl -L https://kuma.io/installer.sh | VERSION=2.9.0 sh -
```

To finish installation we need to add Kuma binaries to path:

```sh
export PATH=$PATH:$(pwd)/kuma-2.9.0/bin
```

```sh
which kuma-cp
```

## Start control plane

Now we need to start control plane in background by running command:

```sh
kuma-cp run
```

## Deploy demo application

### Generate tokens for data plane proxies

On Universal we need to manually create tokens for data plane proxies. To do this need to run this commands (these tokens will be valid
for 30 days):

```sh
kumactl generate dataplane-token --tag kuma.io/service=redis --valid-for=720h > kuma-token-redis
kumactl generate dataplane-token --tag kuma.io/service=demo-app --valid-for=720h > kuma-token-demo-app
```

After generating tokens we can start the data plane proxies that will be used for proxying traffic between `demo-app` and `redis`.

### Start the data plane proxies

Because this is a quickstart, we don't setup certificates for communication
between the data plane proxies and the control plane.
You'll see a warning like the following in the `kuma-dp` logs:

```sql
2024-07-25T20:06:36.082Z	INFO	dataplane	[WARNING] The data plane proxy cannot verify the identity of the control plane because you are not setting the "--ca-cert-file" argument or setting the KUMA_CONTROL_PLANE_CA_CERT environment variable.
```

This isn't related to mTLS between services.

First we can start the data plane proxy for `redis`. On Universal we need to manually create Dataplane resources for data plane proxies, and
run kuma-dp manually, to do this run:

```sh
KUMA_READINESS_PORT=9901 KUMA_APPLICATION_PROBE_PROXY_PORT=9902 kuma-dp run \
  --cp-address=https://localhost:5678/ \
  --dns-enabled=false \
  --dataplane-token-file=kuma-token-redis \
  --dataplane="
  type: Dataplane
  mesh: default
  name: redis
  networking: 
    address: 127.0.0.1
    inbound: 
      - port: 16379
        servicePort: 26379
        serviceAddress: 127.0.0.1
        tags: 
          kuma.io/service: redis
          kuma.io/protocol: tcp
    admin:
      port: 9903"
```

You can notice that we are manually specifying the readiness port with environment variable `KUMA_READINESS_PORT`, when each data plane is
running on separate machines this is not required.

Now we can start the data plane proxy for our demo-app, we can do this by running:

```sh
KUMA_READINESS_PORT=9904 KUMA_APPLICATION_PROBE_PROXY_PORT=9905 kuma-dp run \
  --cp-address=https://localhost:5678/ \
  --dns-enabled=false \
  --dataplane-token-file=kuma-token-demo-app \
  --dataplane="
  type: Dataplane
  mesh: default
  name: demo-app
  networking: 
    address: 127.0.0.1
    outbound:
      - port: 6379
        tags:
          kuma.io/service: redis
    inbound: 
      - port: 15000
        servicePort: 5000
        serviceAddress: 127.0.0.1
        tags: 
          kuma.io/service: demo-app
          kuma.io/protocol: http
    admin:
      port: 9906"
```

### Run kuma-counter-demo app

We will start the kuma-counter-demo in a new terminal window:

1. With the data plane proxies running, we can start our apps, first we will start and configure `Redis`:

```sh
redis-server --port 26379 --daemonize yes && redis-cli -p 26379 set zone local
```

You should see message `OK` from `Redis` if this operation was successful.

2. Now we can start our `demo-app`. To do this we need to download repository with its source code:

```sh
git clone https://github.com/kumahq/kuma-counter-demo.git && cd kuma-counter-demo
```

3. Now we need to run:

```sh
npm install --prefix=app/ && npm start --prefix=app/
```

If `demo-app` was started correctly you will see message:

```sh
Server running on port 5000
```

In a browser, go to [127.0.0.1:5000](http://127.0.0.1:5000) and increment the counter. `demo-app` GUI should work without issues now.

## Explore Kuma GUI

You can view the sidecar proxies that are connected to the Kuma control plane.

Kuma ships with a **read-only** GUI that you can use to retrieve Kuma resources.
By default, the GUI listens on the API port which defaults to `5681`.

To access Kuma we need to navigate to [127.0.0.1:5681/gui](http://127.0.0.1:5681/gui) in your browser.

To learn more, read the documentation about the user interface.

## Introduction to zero-trust security

By default, the network is **insecure and not encrypted**. We can change this with Kuma by enabling
the Mutual TLS policy to provision a Certificate Authority (CA) that
will automatically assign TLS certificates to our services (more specifically to the injected data plane proxies running
alongside the services).

We can enable Mutual TLS with a `builtin` CA backend by executing:

```sh
echo 'type: Mesh
name: default
meshServices:
  mode: Exclusive
mtls:
  enabledBackend: ca-1
  backends:
    - name: ca-1
      type: builtin' | kumactl apply -f -
```

The traffic is now **encrypted and secure**. Kuma does not define default traffic permissions, which
means that no traffic will flow with mTLS enabled until we define a proper MeshTrafficPermission policy.

For now, the demo application won't work.
You can verify this by clicking the increment button again and seeing the error message in the browser.
We can allow the traffic from the `demo-app` to `redis` by applying the following `MeshTrafficPermission`:

```sh
echo 'type: MeshTrafficPermission 
name: allow-from-demo-app
mesh: default 
spec: 
  targetRef: 
    kind: MeshSubset
    tags:
      kuma.io/service: redis
  from: 
    - targetRef: 
        kind: MeshSubset 
        tags:
          kuma.io/service: demo-app
      default: 
        action: Allow' | kumactl apply -f -
```

You can click the increment button, the application should function once again.
However, the traffic to `redis` from any other service than `demo-app` is not allowed.

