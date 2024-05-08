# Simple routing

---

## Deploy a workload

```shell
kubectl apply -f apps/httpbin.yaml
```

---

## Deploy an HTTPRoute to `httpbin`

```yaml linenums="1"
--8<-- "simple-route/httpbin-route.yaml"
```

```shell
kubectl apply -f simple-route/httpbin-route.yaml
```

---

## :white_check_mark: Verify

After the DNS entry is created for the hostname specified in the HTTPRoute, try to [access the app](http://httpbin.esuez.org/):

=== "Using DNS resolution"

    ```shell
    curl http://httpbin.esuez.org/json
    ```

=== "Using `curl` name resolve flag"

    ```shell
    curl http://httpbin.esuez.org/json --resolve httpbin.esuez.org:80:$GATEWAY_IP
    ```

