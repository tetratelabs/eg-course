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

## :white_check_mark: Test it

```shell
export GATEWAY_IP=$(kubectl get gtw eg -o jsonpath='{.status.addresses[0].value}')
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
