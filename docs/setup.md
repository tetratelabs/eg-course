# Installation & setup

## Provision a cluster

```shell
--8<-- "setup/make-local-k3d-cluster"
```

```shell
./setup/make-local-k3d-cluster
```

[About k3d](https://k3d.io/).

---

## Install [EG](https://gateway.envoyproxy.io/)

```shell
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v{{eg.version}} \
  -n envoy-gateway-system --create-namespace
```

---

## Define a [GatewayClass](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/):

```yaml linenums="1"
--8<-- "setup/gateway-class.yaml"
```

```shell
kubectl apply -f setup/gateway-class.yaml
```

---

## Install the `egctl` CLI

Envoy Gateway comes with its own command-line tool, [`egctl`](https://gateway.envoyproxy.io/docs/install/install-egctl/), which provides a variety of helpful subcommands that we explore in subsequent lessons.

`egctl` can be installed in different ways:

- From binaries (published with each release for different platforms).
- Via an install script.
- On MacOS, with the homebrew package manager.

Installation instructions are available in the [Envoy Gateway documentation](https://gateway.envoyproxy.io/docs/install/install-egctl/).