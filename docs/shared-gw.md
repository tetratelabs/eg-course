# Shared Gateway

---

## Deploy the workloads

```shell
kubectl create ns httpbin
```

```shell
kubectl label ns httpbin self-serve-ingress="true"
```

```shell
kubectl apply -f apps/httpbin.yaml -n httpbin
```

Deploy a second workload:

```shell
kubectl create ns customers
```

```shell
kubectl label ns customers self-serve-ingress="true"
```

```shell
kubectl apply -f apps/customers.yaml -n customers
```

```shell
kubectl apply -f apps/web-frontend.yaml -n customers
```

---


## Deploy a [Gateway](https://gateway-api.sigs.k8s.io/api-types/gateway/)

```yaml linenums="1" hl_lines="13-18"
--8<-- "shared-gw/gateway-http.yaml"
```

```shell
kubectl apply -f shared-gw/gateway-http.yaml
```

Wait for the gateway to become available:

```shell
kubectl wait gtw/eg --for=condition=Programmed
```

---

## Configure the routes

```yaml linenums="1"
--8<-- "shared-gw/httpbin-route.yaml"
```

```shell
kubectl apply -f shared-gw/httpbin-route.yaml
```

```yaml linenums="1"
--8<-- "shared-gw/web-frontend-route.yaml"
```

```shell
kubectl apply -f shared-gw/web-frontend-route.yaml
```

---

## :white_check_mark: Verify

Verify that the routes are reachable:

```shell
curl http://httpbin.example.com/json --resolve httpbin.example.com:80:$GATEWAY_IP
```


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
