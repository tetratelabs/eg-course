# Getting started

## Deploy a [Gateway](https://gateway-api.sigs.k8s.io/api-types/gateway/)

```yaml linenums="1"
--8<-- "getting-started/gateway-http.yaml"
```

```shell
kubectl apply -f getting-started/gateway-http.yaml
```

Wait for the gateway to become available:

```shell
kubectl wait gtw/eg --for=condition=Programmed
```

---

## :white_check_mark: Test it

```shell
export GATEWAY_IP=$(kubectl get gtw eg -o jsonpath='{.status.addresses[0].value}')
```

!!! note

    When using a local k3d cluster, k3d will usually configure the loop back address 127.0.0.1 as your gateway IP address.  In that case, configure your GATEWAY_IP as follows:

    ```shell
    export GATEWAY_IP=127.0.0.1
    ```

    On MacOS, there are projects (for example [docker-mac-net-connect](https://github.com/chipmk/docker-mac-net-connect)) that provide a way to connect to containers running on Docker Desktop.


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


---

## Deploy a workload

```shell
kubectl apply -f apps/httpbin.yaml
```

---

## Deploy an HTTPRoute to `httpbin`

```yaml linenums="1"
--8<-- "getting-started/httpbin-route.yaml"
```

```shell
kubectl apply -f getting-started/httpbin-route.yaml
```

---

## :white_check_mark: Verify

After the DNS entry is created for the hostname specified in the HTTPRoute, try to access the app:

```shell
curl http://httpbin.example.com/json --resolve httpbin.example.com:80:$GATEWAY_IP
```

!!! note

    Above, we use `curl`'s `--resolve` flag to resolve the host name to the gateway IP address.  See [here](https://everything.curl.dev/usingcurl/connections/name.html#provide-a-custom-ip-address-for-a-name) for more details.

    It's a simple alternative to creating DNS entries that resolve hostnames to public IP addresses, and allows us to work with a purely local environment.


## Next

In the next section, you will be constructing an ingress configuration that takes into account multiple tenants using a shared gateway.

In anticipation of configuring a shared gateway, delete the simple route and gateway that you built in this scenario:

```shell
kubectl delete httproute httpbin
```

```shell
kubectl delete gtw eg
```