# Shared Gateway

---

Deploy a second workload and a second route associated to the same gateway

```shell
kubectl apply -f apps/customers.yaml -f apps/web-frontend.yaml
```

---

## Configure the route

```yaml linenums="1"
--8<-- "shared-gw/web-frontend-route.yaml"
```

```shell
kubectl apply -f shared-gw/web-frontend-route.yaml
```

---

## :white_check_mark: Verify

After the DNS entry is created, verify that [the route is reachable](http://customers-frontend.example.com/):


=== "Using DNS resolution"

    ```shell
    curl http://customers-frontend.example.com/
    ```

=== "Using `curl` name resolve flag"

    ```shell
    curl http://customers-frontend.example.com/ --resolve customers-frontend.example.com:80:$GATEWAY_IP
    ```

---

## Inspect route configuration

```shell
egctl config envoy-proxy route -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=default \
  -o yaml | bat -l yaml
```

Here is a slightly sanitized copy of the captured output:

```yaml
envoy-gateway-system:
  envoy-default-eg-e41e7b31-59b4dd766f-sf78k:
    dynamicRouteConfigs:
    - routeConfig:
        name: default/eg/http
        virtualHosts:
        - domains:
          - httpbin.example.com
          name: default/eg/http/httpbin_example_com
          routes:
          - match:
              prefix: /
            name: httproute/default/httpbin/rule/0/match/0/httpbin_example_com
            route:
              cluster: httproute/default/httpbin/rule/0
        - domains:
          - customers-frontend.example.com
          name: default/eg/http/customers-frontend_example_com
          routes:
          - match:
              prefix: /
            name: httproute/default/web-frontend/rule/0/match/0/customers-frontend_example_com
            route:
              cluster: httproute/default/web-frontend/rule/0
```
