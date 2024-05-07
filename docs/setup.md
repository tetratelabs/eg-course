# Setup

---

## Provision a cluster

```shell
--8<-- "setup/make-gke-cluster"
```

```shell
./make-gke-cluster
```

---

## Install EG or TEG

=== "Install Envoy Gateway"

    ```shell
    helm install eg oci://docker.io/envoyproxy/gateway-helm \
      --version v0.0.0-latest \
      -n envoy-gateway-system --create-namespace
    ```

=== "Install TEG"

    ```shell
    helm install teg oci://docker.io/tetrate/teg-envoy-gateway-helm \
      --version v1.0.0 \
      -n envoy-gateway-system --create-namespace
    ```

---

## Install `external-dns`

```shell
kubectl apply -f setup/external-dns.yaml
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

## Deploy a [Gateway](https://gateway-api.sigs.k8s.io/api-types/gateway/)

```yaml linenums="1"
--8<-- "setup/gateway-http.yaml"
```

```shell
kubectl apply -f setup/gateway-http.yaml
```

---
