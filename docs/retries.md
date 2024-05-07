# Retries

---

Retries are a great example of how EG extends the Kubernetes Gateway API using [Policy Attachments](https://gateway-api.sigs.k8s.io/reference/policy-attachment/).

---

## Review Gateway-related CRDs

```shell
kubectl api-resources | grep gateway
```

---

## Use [BackendTrafficPolicy](https://gateway.envoyproxy.io/v1.0.1/api/extension_types/#backendtrafficpolicy) to configure retries

```yaml linenums="1"
--8<-- "retries/retry-policy.yaml"
```

```shell
kubectl apply -f retry-policy.yaml
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

```shell
curl --insecure --head https://httpbin.esuez.org/status/500
```

---

## :white_check_mark: Verify

### Tail the gateway logs

```shell
kubectl logs --follow -n envoy-gateway-system deploy envoy-default-eg-
```

TODO: fix the above reference to the gw pod

Prettify JSON log line and note the [Envoy response flag](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags) is URX: UpstreamRetryLimitExceeded.

### Review the proxy configuration

```shell
egctl config envoy-proxy route -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=default \
  -o yaml | bat -l yaml
```

Confirm that it's been updated with the retry rule

## Summary

To configure retries, we had to resort to a BackingTrafficPolicy, an extension to the Gateway API.

In contrast, compare with the [configuration of timeouts](https://gateway.envoyproxy.io/latest/tasks/traffic/http-timeouts/), which is part of the Kubernetes Gateway API specification.
