# Rate limiting

!!! warning

    If during [setup](setup.md) you chose to install Envoy Gateway, then before you proceed with this lab, you will need to deploy Redis, and reconfigure Envoy Gateway with rate limiting pointing to the URL of the Redis instance you deployed.

    Detailed instructions are available [here](https://gateway.envoyproxy.io/v1.0.2/tasks/traffic/global-rate-limit/).


Similar to [retries](retries.md),
rate limiting is not part of the Kubernetes Gateway API specification,
and is configured through Envoy Gateway's [BackendTrafficPolicy](https://gateway.envoyproxy.io/v1.0.2/api/extension_types/#backendtrafficpolicy) resource.

The rate limit is associated with the HTTPRoute you wish to limit.

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

=== "Using DNS resolution"

    ```shell
    for i in {1..4}; do
      curl --insecure --head https://httpbin.esuez.org/
    done
    ```

=== "Using `curl` name resolve flag"

    ```shell
    for i in {1..4}; do
      curl --insecure --head https://httpbin.esuez.org/ --resolve httpbin.esuez.org:443:$GATEWAY_IP
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
  -l gateway.envoyproxy.io/owning-gateway-namespace=default | jq
```

Below is a copy of the prettified JSON log line:

```json linenums="1" hl_lines="6-7"
{
  "start_time": "2024-05-09T00:44:06.909Z",
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
  "duration": "3",
  "x-envoy-upstream-service-time": "-",
  "x-forwarded-for": "172.19.0.4",
  "user-agent": "curl/8.7.1",
  "x-request-id": "a5216e48-3243-42d7-b3f4-8af119efd232",
  ":authority": "httpbin.esuez.org",
  "upstream_host": "-",
  "upstream_cluster": "httproute/default/httpbin/rule/0",
  "upstream_local_address": "-",
  "downstream_local_address": "10.42.0.21:10443",
  "downstream_remote_address": "172.19.0.4:59056",
  "requested_server_name": "httpbin.esuez.org",
  "route_name": "httproute/default/httpbin/rule/0/match/0/httpbin_esuez_org"
}
```

Note the [Envoy response flag](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags) is RL: RateLimited.

## Rate limit distinct users

It is more common for individual users to each have their own limit.

The below example adds a [rate limit selection condition](https://gateway.envoyproxy.io/latest/api/extension_types/#ratelimitselectcondition) to distinguish between users by http header name of `x-user-id`:

```yaml linenums="1" hl_lines="16-19"
--8<-- "ratelimit/distinct-users.yaml"
```

```shell
kubectl apply -f ratelimit/distinct-users.yaml
```

### :white_check_mark: Test it

Sending multiple requests for the same user in succession will produce a result similar to the above simple example:

=== "Using DNS resolution"

    ```shell
    for i in {1..4}; do
      curl --insecure --head -H "x-user-id: eitan" https://httpbin.esuez.org/
    done
    ```

=== "Using `curl` name resolve flag"

    ```shell
    for i in {1..4}; do
      curl --insecure --head -H "x-user-id: eitan" https://httpbin.esuez.org/ \
        --resolve httpbin.esuez.org:443:$GATEWAY_IP
    done
    ```

Following that up with another set of requests from a different user demonstrates that each user has their own, separate rate limiting counter:

=== "Using DNS resolution"

    ```shell
    for i in {1..4}; do
      curl --insecure --head -H "x-user-id: johndoe" https://httpbin.esuez.org/
    done
    ```

=== "Using `curl` name resolve flag"

    ```shell
    for i in {1..4}; do
      curl --insecure --head -H "x-user-id: johndoe" https://httpbin.esuez.org/ \
        --resolve httpbin.esuez.org:443:$GATEWAY_IP
    done
    ```

The curious may wish to inspect the translated configuration at the Envoy proxy:

```shell
egctl config envoy-proxy route -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=default \
  -o yaml | bat -l yaml
```

Here is a sanitized copy of the captured output:

```yaml linenums="1" hl_lines="17-24"
envoy-gateway-system:
  envoy-default-eg-e41e7b31-c7657fcf5-gsgvs:
    dynamicRouteConfigs:
    - ...
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
              rateLimits:
              - actions:
                - genericKey:
                    descriptorKey: httproute/default/httpbin/rule/0/match/0/httpbin_esuez_org
                    descriptorValue: httproute/default/httpbin/rule/0/match/0/httpbin_esuez_org
                - requestHeaders:
                    descriptorKey: rule-0-match-0
                    headerName: x-user-id
```
