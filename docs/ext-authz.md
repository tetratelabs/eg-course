# External Authorization

This scenario demonstrates how Envoy's [external authorization filter](https://www.envoyproxy.io/docs/envoy/v1.30.4/intro/arch_overview/security/ext_authz_filter#arch-overview-ext-authz) can be applied to an HttpRoute.

## Context

We will use the [Ext Authz service sample](https://github.com/istio/istio/tree/master/samples/extauthz) from the Istio distribution.

Deploy the service to the `httpbin` namespace:

```shell
kubectl apply -f ext-authz/ext-authz.yaml -n httpbin
```

## The contract

The service you just deployed will allow (200) any request bearing the header `x-ext-authz: allow`.

The absence of the header, or the header with a value other than `allow` will be denied (403).

## Instructions

Make sure that the `httpbin` service is deployed, and a simple route is defined from the gateway to the service.

Review the following [security policy](https://gateway.envoyproxy.io/docs/api/extension_types/#securitypolicy):

```yaml linenums="1"
--8<-- "ext-authz/security-policy.yaml"
```

Apply the policy:

```shell
kubectl apply -f ext-authz/security-policy.yaml
```

Send a test request:

```shell
curl -v -H "x-ext-authz: allow" http://httpbin.example.com/json --resolve httpbin.example.com:80:$GATEWAY_IP
```

The above request should succeed.

Absence of the header, or header value that is not "allow" will return a 403.

## Discussion

Above, we chose to deploy the `ext-authz` service in the same namespace where our route and security policies reside, so the service was resolved relative to that "local" namespace.

If we had opted to deploy the `ext-authz` service to another namespace, say the `default` namespace, not only would we have had to revise the security policy to reference the service in that namespace (add a `namespace` field to the `backendRef`), but in addition the owner of the target (`default`) namespace would have to give us permission to do so via a [ReferenceGrant](https://gateway-api.sigs.k8s.io/api-types/referencegrant/):

```yaml linenums="1"
--8<-- "ext-authz/ref-grant.yaml"
```

See if you can revise the implementation accordingly.