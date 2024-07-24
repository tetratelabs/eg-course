# WebAssembly

In this scenario we demonstrate how to apply a wasm extension to an HttpRoute.

You will apply an [extension policy](https://gateway.envoyproxy.io/docs/api/extension_types/#envoyextensionpolicy) that references a simple, pre-built wasm plugin to the HttpRoute for the `httpbin` sample application.

The basic logic of the plugin is to inject arbitrary, configurable headers into HTTP responses on the associated route.

## Context

`httpbin` is already deployed to the `default` namespace and the [simple HttpRoute](simple-route.md) is already defined for it.

## Instructions

1. Send a test request to the `httpbin` route:

    ```shell
     curl -v http://httpbin.esuez.org/json --resolve httpbin.esuez.org:80:$GATEWAY_IP
    ```

    The request should succeed, and return some json.

    Note the headers in the HTTP response.

1. Review the following extension policy:

    ```yaml linenums="1"
    --8<-- "wasm/extension-policy.yaml"
    ```

    Apply the policy:

    ```shell
    kubectl apply -f wasm/extension-policy.yaml
    ```

1. Repeat the request:

    ```shell
    curl -v http://httpbin.esuez.org/json --resolve httpbin.esuez.org:80:$GATEWAY_IP
    ```

    Note the response headers contain extra headers from the configuration of the wasm plugin.
