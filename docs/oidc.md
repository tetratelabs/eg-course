# OIDC

We wish to protect the `httpbin` application with OpenID Connect authentication.

For this example, the assumption is that we've pre-configured an openid connect authorization server.

## Configuring Authentication

Before proceeding, configure your issuer URL, client id, and client secret.

For example:

```shell
export ISSUER=""
export CLIENT_ID=""
export CLIENT_SECRET="*****"
```

### Kubernetes secret

Create a Kubernetes secret for the client secret:

```shell
kubectl create secret generic httpbin-client-secret \
  --from-literal=client-secret=$CLIENT_SECRET
```

### Security policy

Review the following security policy specification:

```yaml linenums="1"
--8<-- "oidc/security-policy.yaml"
```

We are basically configuring OIDC authentication for the route for the httpbin application.

Apply the policy, with variable substitution:

```shell
envsubst < oidc/security-policy.yaml | kubectl apply -f -
```

### Test it

1. Visit the [httpbin route](https://httpbin.esuez.org/).
    You will be redirected to your identity provider.
1. Sign in.
    You will be redirected back to the target application.


## Authorization

tbd

## References

- [Envoy Gateway OIDC Authentication Task](https://gateway.envoyproxy.io/latest/tasks/security/oidc/)
