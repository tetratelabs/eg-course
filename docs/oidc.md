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
--8<-- "oidc/oidc-policy.yaml"
```

We are basically configuring OIDC authentication for the route for the httpbin application.

Apply the policy, with variable substitution:

```shell
envsubst < oidc/oidc-policy.yaml | kubectl apply -f -
```

### Test it

1. Visit the [httpbin route](https://httpbin.esuez.org/).
    You will be redirected to your identity provider.
1. Sign in.
    You will be redirected back to the target application.


## Authorization

- Create an administrators group
- Have two users, where only one is a member of the admin group
- Configure the authorization server to set claim "access" to `default` or `privileged` as a function of the user's group membership

Review the following augmented security policy specification:

```yaml linenums="1" hl_lines="19-30"
--8<-- "oidc/security-policy.yaml"
```

We are adding a JWT provider that will populate the header `x-access` with the value from the `access` claim (default or privileged).

Apply the policy, with variable substitution:

```shell
envsubst < oidc/security-policy.yaml | kubectl apply -f -
```

Next, let's expose the "admin" routes only to privileged users:

```yaml linenums="1"
--8<-- "oidc/authz-route.yaml"
```

Above:

- We give access to the exact endpoint (path) `/headers` to all authenticated users.
- For **any other path** (path prefix of /), we match only if the header `x-access` is `privileged`.

Other than the `/headers` (and oauth) endpoints, non-privileged users will not have routes to any of the other endpoints of the `httpbin` application.

Apply the claim-based routing policy:

```shell
kubectl apply -f oidc/authz-route.yaml
```

To test the policy:

- Sign in as a non-privileged user and verify that you can access the `/headers` endpoint but no other `httpbin` application endpoints.
- Sign in as a privileged user and verify that this user has access to all of the `httpbin` app endpoints.


## References

- [Envoy Gateway OIDC Authentication Task](https://gateway.envoyproxy.io/latest/tasks/security/oidc/)
- [JWT claims-based routing](https://gateway.envoyproxy.io/docs/tasks/traffic/http-routing/#jwt-claims-based-routing)
