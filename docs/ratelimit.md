# Rate limiting

Similar to [retries](policy-attachments.md),
rate limiting is not part of the Kubernetes Gateway API specification,
and is configured through Envoy Gateway's [BackendTrafficPolicy](https://gateway.envoyproxy.io/docs/api/extension_types/#backendtrafficpolicy) resource.

The rate limit is associated with the HTTPRoute you wish to limit.

---

## Configuring EG for rate limiting

EG supports Envoy's [Global Rate limiting](https://www.envoyproxy.io/docs/envoy/v1.31.0/intro/arch_overview/other_features/global_rate_limiting.html) feature, which depends on a backing Redis instance to maintain shared state.

Before we can begin to exercise rate limiting, we need to deploy a Redis instance and configure EG with the Redis url.

Detailed instructions are available [in the EG docs](https://gateway.envoyproxy.io/docs/tasks/traffic/global-rate-limit/#prerequisites).

Follow those instructions to:

1. Deploy Redis
1. Update the `envoy-gateway-config` ConfigMap to specify the Redis backend url
1. Restart the `envoy-gateway` deployment

---

## Simple example

Configure access to `httpbin` to be limited to three requests per minute:

```yaml linenums="1"
--8<-- "ratelimit/simple.yaml"
```

```shell
kubectl apply -f ratelimit/simple.yaml
```

### :white_check_mark: Test it

Send four requests in succession, the fourth should be rate-limited:

```shell
for i in {1..4}; do
  curl --insecure --head https://httpbin.example.com/ --resolve httpbin.example.com:443:$GATEWAY_IP
done
```

Here is the captured output:

```console
HTTP/2 200
server: gunicorn/19.9.0
date: Tue, 07 May 2024 22:33:12 GMT
content-type: text/html; charset=utf-8
content-length: 9593
access-control-allow-origin: *
access-control-allow-credentials: true
x-ratelimit-limit: 3, 3;w=60
x-ratelimit-remaining: 2
x-ratelimit-reset: 48

HTTP/2 200
server: gunicorn/19.9.0
date: Tue, 07 May 2024 22:33:12 GMT
content-type: text/html; charset=utf-8
content-length: 9593
access-control-allow-origin: *
access-control-allow-credentials: true
x-ratelimit-limit: 3, 3;w=60
x-ratelimit-remaining: 1
x-ratelimit-reset: 48

HTTP/2 200
server: gunicorn/19.9.0
date: Tue, 07 May 2024 22:33:12 GMT
content-type: text/html; charset=utf-8
content-length: 9593
access-control-allow-origin: *
access-control-allow-credentials: true
x-ratelimit-limit: 3, 3;w=60
x-ratelimit-remaining: 0
x-ratelimit-reset: 48

HTTP/2 429
x-envoy-ratelimited: true
x-ratelimit-limit: 3, 3;w=60
x-ratelimit-remaining: 0
x-ratelimit-reset: 48
date: Tue, 07 May 2024 22:33:12 GMT
```

Above, note the `x-ratelimit-*` headers that inform us of the limit, the number of requests remaining, and the amount of time (in seconds) until the corresponding counter is reset.

---

## :white_check_mark: Verify: Tail the gateway logs

```shell
kubectl logs --tail 1 -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=infra | jq
```

Below is a copy of the prettified JSON log line:

```json linenums="1" hl_lines="6-7"
{
  "start_time": "2024-08-08T00:21:14.929Z",
  "method": "HEAD",
  "x-envoy-origin-path": "/",
  "protocol": "HTTP/2",
  "response_code": "429",
  "response_flags": "RL",
  "response_code_details": "request_rate_limited",
  "connection_termination_details": "-",
  "upstream_transport_failure_reason": "-",
  "bytes_received": "0",
  "bytes_sent": "0",
  "duration": "1",
  "x-envoy-upstream-service-time": "-",
  "x-forwarded-for": "172.19.0.1",
  "user-agent": "curl/8.9.1",
  "x-request-id": "04186347-7203-41d0-8b56-965bc32ef60a",
  ":authority": "httpbin.example.com",
  "upstream_host": "-",
  "upstream_cluster": "httproute/httpbin/httpbin/rule/0",
  "upstream_local_address": "-",
  "downstream_local_address": "10.42.0.22:10443",
  "downstream_remote_address": "172.19.0.1:62217",
  "requested_server_name": "httpbin.example.com",
  "route_name": "httproute/httpbin/httpbin/rule/0/match/0/httpbin_example_com"
}
```

Note the [Envoy response flag](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags) is RL: RateLimited.

## Rate limit distinct users

It is more common for individual users to each have their own limit.

The below example adds a [rate limit selection condition](https://gateway.envoyproxy.io/docs/api/extension_types/#ratelimitselectcondition) to distinguish between users by http header name of `x-user-id`:

```yaml linenums="1" hl_lines="16-19"
--8<-- "ratelimit/distinct-users.yaml"
```

```shell
kubectl apply -f ratelimit/distinct-users.yaml
```

### :white_check_mark: Test it

Sending multiple requests for the same user in succession will produce a result similar to the above simple example:

```shell
for i in {1..4}; do
  curl --insecure --head -H "x-user-id: eitan" https://httpbin.example.com/ \
    --resolve httpbin.example.com:443:$GATEWAY_IP
done
```

Following that up with another set of requests from a different user demonstrates that each user has their own, separate rate limiting counter:

```shell
for i in {1..4}; do
  curl --insecure --head -H "x-user-id: johndoe" https://httpbin.example.com/ \
    --resolve httpbin.example.com:443:$GATEWAY_IP
done
```

The curious may wish to inspect the translated configuration at the Envoy proxy:

```shell
egctl config envoy-proxy route -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=infra \
  -o yaml | bat -l yaml
```

Here is a sanitized copy of the captured output:

```yaml linenums="1" hl_lines="17-24"
envoy-gateway-system:
  envoy-infra-eg-eade8e06-5f9f47c8bf-gwh4g:
    dynamicRouteConfigs:
    - ...
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
              rateLimits:
              - actions:
                - genericKey:
                    descriptorKey: httproute/httpbin/httpbin/rule/0/match/0/httpbin_example_com
                    descriptorValue: httproute/httpbin/httpbin/rule/0/match/0/httpbin_example_com
                - requestHeaders:
                    descriptorKey: rule-0-match-0
                    headerName: x-user-id
```
