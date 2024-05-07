# Authentication

Envoy Gateway supports a number of [distinct authentication mechanisms](https://gateway.envoyproxy.io/latest/api/extension_types/#securitypolicyspec), including JWT, OIDC, external authorization, and basic auth.

In this exercise, we keep things simple and demonstrate basic auth.

Once more, we are dealing with a feature that is outside the current Gateway API specification, and so we use a [SecurityPolicy](https://gateway.envoyproxy.io/v1.0.1/api/extension_types/#securitypolicy) attachment against the route we wish to protect, which in this case is the `httpbin` route.

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

    ```shell
    curl --insecure --head https://httpbin.esuez.org/
    ```

    ```console
    HTTP/2 401
    content-length: 58
    content-type: text/plain
    date: Tue, 07 May 2024 23:18:11 GMT
    ```

1. Authenticated requests succeed:

    ```shell
    curl --insecure --head --user eitan:correctpassword https://httpbin.esuez.org/
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

    ```shell
    curl --insecure --head --user eitan:wrongpassword https://httpbin.esuez.org/
    ```

    ```console
    HTTP/2 401
    content-length: 66
    content-type: text/plain
    date: Tue, 07 May 2024 23:18:23 GMT
    ```
