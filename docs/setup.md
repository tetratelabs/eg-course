# Setup

## Artifacts

Download all yaml artifacts referenced in all scenarios as a single .tgz file [here](artifacts.tgz).

## Provision a cluster

===+ "On GCP"

    ```shell
    --8<-- "setup/make-gke-cluster"
    ```

    ```shell
    ./setup/make-gke-cluster
    ```

=== "Locally with k3d"

    ```shell
    k3d cluster create my-k8s-cluster \
      --k3s-arg "--disable=traefik@server:0" \
      --port 80:80@loadbalancer \
      --port 443:443@loadbalancer
    ```

    [About k3d](https://k3d.io/v5.6.3/).

---

## Install [EG](https://gateway.envoyproxy.io/) or [TEG](https://docs.tetrate.io/envoy-gateway/)

TEG installs Redis and the Envoy rate limit service, meaning that it's pre-configured for rate-limiting.

=== "Install Envoy Gateway"

    ```shell
    helm install eg oci://docker.io/envoyproxy/gateway-helm \
      --version v0.0.0-latest \
      -n envoy-gateway-system --create-namespace
    ```

===+ "Install TEG"

    ```shell
    helm install teg oci://docker.io/tetrate/teg-envoy-gateway-helm \
      --version v1.0.0 \
      -n envoy-gateway-system --create-namespace
    ```

    Study the deployments in `envoy-gateway-system`:

    ```shell
    kubectl get deploy -n envoy-gateway-system
    ```

    See [architecture](https://docs.tetrate.io/envoy-gateway/introduction/architecture).

---

## Install [`external-dns`](https://kubernetes-sigs.github.io/external-dns/)

A convenience that automatically configures DNS for routes.

!!! warning

    This will not work locally, and requires edits to point to your DNS zone and provider.

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

## :white_check_mark: Test it

===+ "Obtain Gateway IP"

    ```shell
    export GATEWAY_IP=$(kubectl get gtw eg -o jsonpath='{.status.addresses[0].value}')
    ```

=== "Using a local k3d cluster"

    For `k3d`, use 127.0.0.1 as your GATEWAY_IP

    ```shell
    export GATEWAY_IP=127.0.0.1
    ```

```shell
curl -v http://$GATEWAY_IP/
```

```console linenums="1" hl_lines="9"
*   Trying 34.121.222.176:80...
* Connected to 34.121.222.176 (34.121.222.176) port 80
> GET / HTTP/1.1
> Host: 34.121.222.176
> User-Agent: curl/8.7.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 404 Not Found
< date: Tue, 07 May 2024 23:39:35 GMT
< content-length: 0
<
* Connection #0 to host 34.121.222.176 left intact
```

Why do we get a 404 (Not Found)?
