# OIDC with Okta

We wish to protect the `httpbin` application with OpenID Connect authentication.

Once authentication is established, we demonstrate how to configure authorization policies for routes based on JWT claims from the user's access token.

This example uses Okta as the identity provider where we define our users and their level of access to the application.

## Okta configuration

You will configure two users, one will be a default user with standard access to the application, the other will be a privileged user with access to additional endpoints in the application.

### Configure Groups

Create a group named _admin_.

In Okta:

- Navigate to _Directory_ and then to _Groups_.
- Click the button _Add group_.
- Enter the group name.

### Configure Users

Create a user with username _johndoe@example.com_ and another with username _zeboss@example.com_.

In Okta:

- Navigate to _Directory_ and then to _People_.
- Click the button _Add person_.
- Enter the details:  user type (User), first and last names, the username (above email address).
- Check _I will set the password_.
- Enter a password.
- Uncheck _User must change password on first login_.
- For the _zeboss_ user, also associate them to the group _admin_ via the _Groups_ field.
- Click _Save_.

### Configure the Application

The following instructions are for the hostname `httpbin.esuez.org`.

Edit this value for your specific domain.

In Okta:

- Navigate to _Applications_ and then to _Application_.
- Click the button _Create App Integration_.
- Select _OIDC_ as the sign-in method, and _Web Application_ for the application type, and click _Next_.
- For _App integration name_, enter `httpbin.esuez.org`.
- Specify `https://httpbin.esuez.org/oauth2/callback` for the _Sign-in redirect URIs_ field.
- Enter `https://httpbin.esuez.org/logout` for the _Sign-out redirect URIs_ field.
- Click _Save_.

Edit the application as follows:

- Select the tab named _Assignments_.
- Click the _Assign_ button.
- Select _Assign to People_ and assign both users to the application.
- Select _Assign to Groups_ and assign the admin group to the application.

In the _General_ tab, make note of the client id and client secret.
You will need both to configure the environment variables CLIENT_ID and CLIENT_SECRET.

### Configure the Authorization Server

In Okta:

- Navigate to _Security_ and then to _API_.
- Under _Authorization Servers_ select the existing _default_ authorization server.
- Make note of the issuer URL to set the ISSUER environment variable.
- Select the _Claims_ tab.
- Click the button _Add Claim_.
- Set the claim name to _access_.
- Keep the defaults: include in token type "Access Token", and value type of "Expression".
- For the expression value, enter:  `	(user == null) ? "default" : user.isMemberOfGroupName("admin") ? "privileged" : "default"`
- Click _Create_.

The above expression translates _admin_ group membership to the value "privileged"; absence of membership results in the value "default."

### Test the token

- Select the tab _Token Preview_
- Select the httpbin app integration, the grant type of authorization code, one of the two users, the the scope named _openid_
- Select _Preview Token_
- Select the tab named _token_ (that would be the access token)
- Verify that the payload has a claim named _access_ with value of _privileged_ for the admin user and _default_ for the other user.

## Configuring Authentication

Before proceeding, configure your issuer URL, client id, and client secret.

For example:

```shell
export ISSUER="https://dev-*****.okta.com/oauth2/default"
export CLIENT_ID="..."
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

Review the following augmented security policy specification:

```yaml linenums="1" hl_lines="20-31"
--8<-- "oidc/security-policy.yaml"
```

We are adding a JWT provider that will populate the header `x-access` with the value from the `access` claim (default or privileged).

Apply the policy, with variable substitution:

```shell
envsubst < oidc/security-policy.yaml | kubectl apply -f -
```

Next, expose the "admin" routes only to privileged users:

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

- [Envoy Gateway OIDC Authentication Task](https://gateway.envoyproxy.io/docs/tasks/security/oidc/)
- [JWT claims-based routing](https://gateway.envoyproxy.io/docs/tasks/traffic/http-routing/#jwt-claims-based-routing)
