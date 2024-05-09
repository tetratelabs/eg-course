# Retries

Retries are an example of how EG extends the Kubernetes Gateway API using [Policy Attachments](https://gateway-api.sigs.k8s.io/reference/policy-attachment/).

---

## Review Gateway-related CRDs

```shell
kubectl api-resources | grep gateway
```

Here is a slightly sanitized copy of the captured output:

```console linenums="1" hl_lines="1-5"
gateway.envoyproxy.io/v1alpha1       BackendTrafficPolicy
gateway.envoyproxy.io/v1alpha1       ClientTrafficPolicy
gateway.envoyproxy.io/v1alpha1       EnvoyPatchPolicy
gateway.envoyproxy.io/v1alpha1       EnvoyProxy
gateway.envoyproxy.io/v1alpha1       SecurityPolicy
gateway.networking.k8s.io/v1alpha2   BackendTLSPolicy
gateway.networking.k8s.io/v1         GatewayClass
gateway.networking.k8s.io/v1         Gateway
gateway.networking.k8s.io/v1alpha2   GRPCRoute
gateway.networking.k8s.io/v1         HTTPRoute
gateway.networking.k8s.io/v1beta1    ReferenceGrant
gateway.networking.k8s.io/v1alpha2   TCPRoute
gateway.networking.k8s.io/v1alpha2   TLSRoute
gateway.networking.k8s.io/v1alpha2   UDPRoute
```

---

## Use [BackendTrafficPolicy](https://gateway.envoyproxy.io/v1.0.1/api/extension_types/#backendtrafficpolicy) to configure retries

```yaml linenums="1" hl_lines="12-24"
--8<-- "retries/httpbin-policy.yaml"
```

```shell
kubectl apply -f retries/httpbin-policy.yaml
```

---

## Review the proxy's "stats"

Specifically, the `envoy_cluster_upstream_rq_retry` metric:

```shell
watch 'egctl x stats envoy-proxy -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=default \
  | grep "envoy_cluster_upstream_rq_retry{envoy_cluster_name=\"httproute/default/httpbin/rule/0\"}"'
```

In another terminal, call a failing endpoint


=== "Using DNS resolution"

    ```shell
    curl --insecure --head https://httpbin.esuez.org/status/500
    ```

=== "Using `curl` name resolve flag"

    ```shell
    curl --insecure --head https://httpbin.esuez.org/status/500 \
      --resolve httpbin.esuez.org:443:$GATEWAY_IP
    ```

---

## :white_check_mark: Verify: Tail the gateway logs

```shell
kubectl logs --tail 1 -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=default | jq
```

Below is a copy of the prettified JSON log line:

```json linenums="1" hl_lines="7"
{
    "start_time": "2024-05-07T21:28:06.447Z",
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
    "duration": "1837",
    "x-envoy-upstream-service-time": "-",
    "x-forwarded-for": "136.49.247.103",
    "user-agent": "curl/8.7.1",
    "x-request-id": "f8d9ee84-0f3b-4bc8-a8b7-a023226704b9",
    ":authority": "httpbin.esuez.org",
    "upstream_host": "10.48.2.12:8080",
    "upstream_cluster": "httproute/default/httpbin/rule/0",
    "upstream_local_address": "10.48.0.13:36898",
    "downstream_local_address": "10.48.0.13:10443",
    "downstream_remote_address": "136.49.247.103:52999",
    "requested_server_name": "httpbin.esuez.org",
    "route_name": "httproute/default/httpbin/rule/0/match/0/httpbin_esuez_org"
}
```

Note the [Envoy response flag](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags) is URX: UpstreamRetryLimitExceeded.

## :white_check_mark: Verify: Review the proxy configuration

```shell
egctl config envoy-proxy route -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=default \
  -o yaml | bat -l yaml
```

Confirm that the routing configuration has been updated with the retry rule.

Here is a sanitized copy of the captured output:

```yaml linenums="1" hl_lines="17-28"
envoy-gateway-system:
  envoy-default-eg-e41e7b31-c7657fcf5-gsgvs:
    dynamicRouteConfigs:
    ...
    - routeConfig:
        name: default/eg/https
        virtualHosts:
        - domains:
          - httpbin.esuez.org
          name: default/eg/https/httpbin_esuez_org
          routes:
          - match:
              prefix: /
            name: httproute/default/httpbin/rule/0/match/0/httpbin_esuez_org
            route:
              cluster: httproute/default/httpbin/rule/0
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



## Summary

To configure retries, we had to resort to a BackingTrafficPolicy, an extension to the Gateway API.
In contrast, compare with [timeouts](https://gateway-api.sigs.k8s.io/api-types/httproute/?h=#timeouts-optional),
which are configured directly on the HTTPRoute resource, since timeouts are a part of the Kubernetes Gateway API specification.
