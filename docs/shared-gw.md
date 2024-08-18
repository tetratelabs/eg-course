# Shared Gateways

One important aspect of the Kubernetes Gateway API is its ability to accommodate multiple personas.  Cluster operators typically provision and control gateways, but application teams want the ability self-service routes to their backend applications.

In this scenario, we explore two teams each deploying distinct applications to separate namespaces who have the ability to specify their own routes against a shared gateway managed by the operations team.

---

## Design and conventions

The platform team decides to deploy their gateway to the `infra` namespace, and defines a convention whereby the ability to attach routes to the gateway can be given to application teams when their application namespaces are labeled with `self-serve-ingress=true`.

---

## Deploy the [Gateway](https://gateway-api.sigs.k8s.io/api-types/gateway/)

Create the namespace:

```shell
kubectl create ns infra
```

Provision the gateway:

```yaml linenums="1" hl_lines="13-18"
--8<-- "shared-gw/gateway-http.yaml"
```

Above, note the `allowedRoutes` section which allows the attachment of routes from namespaces with the corresponding label.

```shell
kubectl apply -f shared-gw/gateway-http.yaml
```

Wait for the gateway to become available:

```shell
kubectl wait gtw/eg -n infra --for=condition=Programmed
```

### Capture the Gateway IP address

```shell
export GATEWAY_IP=$(kubectl get gtw eg -n infra -o jsonpath='{.status.addresses[0].value}')
```

## Deploy the workloads

### The `httpbin` app

First, create and label the namespace (we assume RBAC is in place to allow only operators to do this):

```shell
kubectl create ns httpbin
```

```shell
kubectl label ns httpbin self-serve-ingress="true"
```

Next, deploy the application:

```shell
kubectl apply -f apps/httpbin.yaml -n httpbin
```

### The `customers` app

Create the `customers` namespace:

```shell
kubectl create ns customers
```

```shell
kubectl label ns customers self-serve-ingress="true"
```

This app consists of two separate deployments:  the `customers` service and the `web-frontend`:

```shell
kubectl apply -f apps/customers.yaml -n customers
```

```shell
kubectl apply -f apps/web-frontend.yaml -n customers
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

## Inspect route configuration

Review the `status` section of the shared gateway (named `eg`):

```shell
kubectl get -n infra gtw eg -o yaml
```

Under `listeners`, verify that the number of attached routes is two (2).

Next, use `egctl` to inspect the `route` section of the Envoy configuration:

```shell
egctl config envoy-proxy route -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=infra \
  -o yaml
```

Here is a slightly sanitized copy of the captured output:

```yaml
envoy-gateway-system:
  envoy-infra-eg-eade8e06-84c7dd49d-rwdpk:
    dynamicRouteConfigs:
    - routeConfig:
        name: infra/eg/http
        virtualHosts:
        - domains:
          - httpbin.example.com
          name: infra/eg/http/httpbin_example_com
          routes:
          - match:
              prefix: /
            name: httproute/httpbin/httpbin/rule/0/match/0/httpbin_example_com
            route:
              cluster: httproute/httpbin/httpbin/rule/0
        - domains:
          - customers-frontend.example.com
          name: infra/eg/http/customers-frontend_example_com
          routes:
          - match:
              prefix: /
            name: httproute/customers/web-frontend/rule/0/match/0/customers-frontend_example_com
            route:
              cluster: httproute/customers/web-frontend/rule/0
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

## Summary

With the Kubernetes Gateway API, have can achieve a separation of concerns, where the platform team can manage a shared gateway, while at the same time application teams can (with the permission of the platform team) define the routes for their applications.
