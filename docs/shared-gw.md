# Shared Gateway

---

Deploy a second workload and a second route associated to the same gateway

```shell
kubectl apply -f apps/customers.yaml -f apps/web-frontend.yaml
```

---

## Configure the route

```shell
kubectl apply -f web-frontend-route.yaml
```

```yaml linenums="1"
--8<-- "shared-gw/web-frontend-route.yaml"
```

---

## :white_check_mark: Verify

After the DNS entry is created, verify that the route is reachable:

```shell
curl http://customers-frontend.esuez.org/
```

---

## Inspect route configuration

```shell
egctl config envoy-proxy route -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=default \
  -o yaml | bat -l yaml
```

TODO: capture console output to show Envoy route config
