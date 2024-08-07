# Policy attachments

Retries are an example of how EG extends the Kubernetes Gateway API using [Policy Attachments](https://gateway-api.sigs.k8s.io/reference/policy-attachment/).

To understand policy attachments, we begin with an example of a feature that is part of the Kubernetes Gateway API:  [timeouts](https://gateway-api.sigs.k8s.io/api-types/httproute/?h=#timeouts-optional), and then turn our attention to [retries](https://gateway.envoyproxy.io/docs/tasks/traffic/retry/).

---

## Timeouts

The Kubernetes Gateway API supports configuring of timeouts.

We can specify a total request timeout on the `httpbin` route of one second, as follows:

```yaml linenums="1" hl_lines="17-18"
--8<-- "policy-attachments/httpbin-route.yaml"
```

An HTTPS request to the `httpbin` application will continue to function:

```shell
curl --insecure https://httpbin.example.com/json \
  --resolve httpbin.example.com:443:$GATEWAY_IP
```

On the other hand, targeting an endpoint with a delay of two (2) seconds results in a request timeout (504):

```shell
curl --insecure https://httpbin.example.com/delay/2 \
  --resolve httpbin.example.com:443:$GATEWAY_IP
```

```console
upstream request timeout
```

Configuring timeouts was simple an easy.

Envoy is a proxy with more features than the Kubernetes Gateway API codifies.  The API however does provide guidance on how to formally extend its API via policy attachments.

Let us look at how Envoy Gateway utilizes policy attachments to expose other features of the Envoy proxy.

---

## Review Gateway-related CRDs

Inspect the gateway-related CRDs defined in your cluster:

```shell
kubectl api-resources | grep gateway
```

Here is a slightly sanitized copy of the captured output:

```console linenums="1" hl_lines="1-7"
gateway.envoyproxy.io/v1alpha1       Backend
gateway.envoyproxy.io/v1alpha1       BackendTrafficPolicy
gateway.envoyproxy.io/v1alpha1       ClientTrafficPolicy
gateway.envoyproxy.io/v1alpha1       EnvoyExtensionPolicy
gateway.envoyproxy.io/v1alpha1       EnvoyPatchPolicy
gateway.envoyproxy.io/v1alpha1       EnvoyProxy
gateway.envoyproxy.io/v1alpha1       SecurityPolicy
gateway.networking.k8s.io/v1alpha2   BackendLBPolicy
gateway.networking.k8s.io/v1alpha3   BackendTLSPolicy
gateway.networking.k8s.io/v1         GatewayClass
gateway.networking.k8s.io/v1         Gateway
gateway.networking.k8s.io/v1         GRPCRoute
gateway.networking.k8s.io/v1         HTTPRoute
gateway.networking.k8s.io/v1beta1    ReferenceGrant
gateway.networking.k8s.io/v1alpha2   TCPRoute
gateway.networking.k8s.io/v1alpha2   TLSRoute
gateway.networking.k8s.io/v1alpha2   UDPRoute
```

Note the standard [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/reference/spec/) CRDs as well as [additional ones defined by the Envoy Gateway project](https://gateway.envoyproxy.io/docs/api/extension_types/).

---

## Use [BackendTrafficPolicy](https://gateway.envoyproxy.io/docs/api/extension_types/#backendtrafficpolicy) to configure retries

```yaml linenums="1" hl_lines="12-24"
--8<-- "policy-attachments/httpbin-policy.yaml"
```

```shell
kubectl apply -f policy-attachments/httpbin-policy.yaml
```

---

## :white_check_mark: Verify: Review the proxy configuration

```shell
egctl config envoy-proxy route -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=infra \
  -o yaml | bat -l yaml
```

Confirm that the routing configuration has been updated with the retry rule.

Here is a sanitized copy of the captured output:

```yaml linenums="1" hl_lines="17-28"
envoy-gateway-system:
  envoy-infra-eg-eade8e06-5f9f47c8bf-gwh4g:
    dynamicRouteConfigs:
    ...
    - routeConfig:
        name: infra/eg/https-httpbin
        virtualHosts:
        - domains:
          - httpbin.example.com
          name: infra/eg/https-httpbin/httpbin_example_com
          routes:
          - match:
              prefix: /
            name: httproute/httpbin/httpbin/rule/0/match/0/httpbin_example_com
            route:
              cluster: httproute/httpbin/httpbin/rule/0
              retryPolicy:
                hostSelectionRetryMaxAttempts: "5"
                numRetries: 5
                perTryTimeout: 0.250s
                retriableStatusCodes:
                - 500
                retryBackOff:
                  baseInterval: 0.100s
                  maxInterval: 10s
                retryHostPredicate:
                - name: envoy.retry_host_predicates.previous_hosts
                retryOn: connect-failure,retriable-status-codes
```

---

## :white_check_mark: Verify:  Review the proxy's "stats"

Specifically, the `envoy_cluster_upstream_rq_retry` metric:

```shell
watch 'egctl x stats envoy-proxy -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=infra \
  | grep "envoy_cluster_upstream_rq_retry{envoy_cluster_name=\"httproute/httpbin/httpbin/rule/0\"}"'
```

In another terminal, call a failing endpoint:

```shell
curl --insecure --head https://httpbin.example.com/status/500 \
  --resolve httpbin.example.com:443:$GATEWAY_IP
```

---

Another convenient way to get at the stats exposed by the Envoy proxy is through the [Envoy admin interface](https://www.envoyproxy.io/docs/envoy/latest/operations/admin):

```shell
egctl x dashboard envoy-proxy -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=infra
```

Click on the `stats` endpoint and look for metrics with "retry" in their name.

---

## :white_check_mark: Verify: Tail the gateway logs

```shell
kubectl logs --tail 1 -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=infra | jq
```

Below is a copy of the prettified JSON log line:

```json linenums="1" hl_lines="7"
{
  "start_time": "2024-08-07T23:32:00.874Z",
  "method": "HEAD",
  "x-envoy-origin-path": "/status/500",
  "protocol": "HTTP/2",
  "response_code": "500",
  "response_flags": "URX",
  "response_code_details": "via_upstream",
  "connection_termination_details": "-",
  "upstream_transport_failure_reason": "-",
  "bytes_received": "0",
  "bytes_sent": "0",
  "duration": "787",
  "x-envoy-upstream-service-time": "-",
  "x-forwarded-for": "172.19.0.1",
  "user-agent": "curl/8.9.1",
  "x-request-id": "fff65b4e-5a41-41fb-8a3e-fc9f33121059",
  ":authority": "httpbin.example.com",
  "upstream_host": "10.42.0.12:8080",
  "upstream_cluster": "httproute/httpbin/httpbin/rule/0",
  "upstream_local_address": "10.42.0.22:42978",
  "downstream_local_address": "10.42.0.22:10443",
  "downstream_remote_address": "172.19.0.1:60922",
  "requested_server_name": "httpbin.example.com",
  "route_name": "httproute/httpbin/httpbin/rule/0/match/0/httpbin_example_com"
}
```

Note the [Envoy response flag](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags) is URX: UpstreamRetryLimitExceeded.


## Summary

To configure retries, we had to resort to using a BackingTrafficPolicy, an extension to the Gateway API.
In contrast, compare with [timeouts](https://gateway-api.sigs.k8s.io/api-types/httproute/?h=#timeouts-optional),
which are configured directly on the HTTPRoute resource, since timeouts are a part of the Kubernetes Gateway API specification.
