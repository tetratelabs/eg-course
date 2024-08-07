# Setup

## Provision a cluster

===+ "Locally with k3d"

    ```shell
    --8<-- "setup/make-local-k3d-cluster"
    ```

    ```shell
    ./setup/make-local-k3d-cluster
    ```

    [About k3d](https://k3d.io/).

=== "On GCP"

    ```shell
    --8<-- "setup/make-gke-cluster"
    ```

    ```shell
    ./setup/make-gke-cluster
    ```

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