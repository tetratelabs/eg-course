# External Authorization

This scenario demonstrates how Envoy's [external authorization filter](https://www.envoyproxy.io/docs/envoy/v1.30.4/intro/arch_overview/security/ext_authz_filter#arch-overview-ext-authz) can be applied to an HttpRoute.

## Context

We will use the [Ext Authz service sample](https://github.com/istio/istio/tree/master/samples/extauthz) from the Istio distribution.

Deploy the service:

```shell
kubectl apply -f ext-authz/ext-authz.yaml
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
curl -v -H "x-ext-authz: allow" http://httpbin.esuez.org/json --resolve httpbin.esuez.org:80:$GATEWAY_IP
```

The above request should succeed.

Absence of the header, or header value that is not "allow" will return a 403.
