# Authentication

Envoy Gateway supports a number of [distinct authentication mechanisms](https://gateway.envoyproxy.io/docs/api/extension_types/#securitypolicyspec), including JWT, OIDC, external authorization, and basic auth.

In this exercise, we keep things simple and demonstrate basic auth.

Once more, we are dealing with a feature that is outside the current Gateway API specification, and so we use a [SecurityPolicy](https://gateway.envoyproxy.io/docs/api/extension_types/#securitypolicy) attachment against the route we wish to protect, which in this case is the `httpbin` route.

```yaml linenums="1" hl_lines="13"
--8<-- "auth/basic.yaml"
```

The value of the `name` field on line 13 refers a secret containing an [`htpasswd`](https://httpd.apache.org/docs/2.4/programs/htpasswd.html) file.

1. Create the htpasswd file:

    ```shell
    htpasswd -cs .htpasswd eitan
    ```

1. Create the secret from the generated file:

    ```shell
    kubectl create secret generic httpbin-users --from-file=.htpasswd
    ```

1. Apply the security policy:

    ```shell
    kubectl apply -f auth/basic.yaml
    ```

## :white_check_mark: Test it

[Access the application](https://httpbin.esuez.org/), or:

1. Request without credentials return a 401 (Forbidden)

    === "Using DNS resolution"

        ```shell
        curl --insecure --head https://httpbin.esuez.org/
        ```

    === "Using `curl` name resolve flag"

        ```shell
        curl --insecure --head https://httpbin.esuez.org/ \
          --resolve httpbin.esuez.org:443:$GATEWAY_IP
        ```

    ```console
    HTTP/2 401
    content-length: 58
    content-type: text/plain
    date: Tue, 07 May 2024 23:18:11 GMT
    ```

1. Authenticated requests succeed:

    === "Using DNS resolution"

        ```shell
        curl --insecure --head --user eitan:correctpassword https://httpbin.esuez.org/
        ```

    === "Using `curl` name resolve flag"

        ```shell
        curl --insecure --head --user eitan:correctpassword https://httpbin.esuez.org/ \
          --resolve httpbin.esuez.org:443:$GATEWAY_IP
        ```

    ```console
    HTTP/2 200
    server: gunicorn/19.9.0
    date: Tue, 07 May 2024 23:18:20 GMT
    content-type: text/html; charset=utf-8
    content-length: 9593
    access-control-allow-origin: *
    access-control-allow-credentials: true
    ```

1. Bad credentials produce a 401 (Forbidden):

    === "Using DNS resolution"

        ```shell
        curl --insecure --head --user eitan:wrongpassword https://httpbin.esuez.org/
        ```

    === "Using `curl` name resolve flag"

        ```shell
        curl --insecure --head --user eitan:wrongpassword https://httpbin.esuez.org/ \
          --resolve httpbin.esuez.org:443:$GATEWAY_IP
        ```

    ```console
    HTTP/2 401
    content-length: 66
    content-type: text/plain
    date: Tue, 07 May 2024 23:18:23 GMT
    ```

## Inspect the Proxy configuration

We can look at the Envoy listeners configuration and inspect the HTTP connection manager's filter chain to confirm that the [basic auth filter](https://www.envoyproxy.io/docs/envoy/v1.30.1/configuration/http/http_filters/basic_auth_filter.html) is installed.

```shell
egctl config envoy-proxy listener -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=default \
  -o yaml | bat -l yaml
```

Here is sanitized output for the HTTPS listener:

```yaml linenums="1" hl_lines="10-15"
...
filterChains:
- filterChainMatch:
    serverNames:
    - httpbin.esuez.org
  filters:
  - name: envoy.filters.network.http_connection_manager
    typedConfig:
      httpFilters:
      - name: envoy.filters.http.basic_auth_httproute/default/httpbin/rule/0/match/0/httpbin_esuez_org
        disabled: true
        typedConfig:
          '@type': type.googleapis.com/envoy.extensions.filters.http.basic_auth.v3.BasicAuth
          users:
            inlineBytes: W3JlZGFjdGVkXQ==
      - name: envoy.filters.http.ratelimit
        typedConfig:
          '@type': type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
          domain: default/eg/https
          enableXRatelimitHeaders: DRAFT_VERSION_03
          rateLimitService:
            grpcService:
              envoyGrpc:
                clusterName: ratelimit_cluster
            transportApiVersion: V3
      - name: envoy.filters.http.router
        typedConfig:
          '@type': type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
          suppressEnvoyHeaders: true
  ...
```

The basic auth filter is configured at the level of the listener, but disabled there.
Since the authentication rule is applied at the route level, we need to look at the route.

```shell
egctl config envoy-proxy route -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=default \
  -o yaml | bat -l yaml
```

Here is the salient part of the output:

```yaml linenums="1" hl_lines="19-23"
envoy-gateway-system:
  envoy-default-eg-e41e7b31-c7657fcf5-tmsdt:
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
              rateLimits:
              ...
            typedPerFilterConfig:
              envoy.filters.http.basic_auth:
                '@type': type.googleapis.com/envoy.extensions.filters.http.basic_auth.v3.BasicAuthPerRoute
                users:
                  inlineBytes: W3JlZGFjdGVkXQ==
```
