# HTTPS

The objective of this scenario is to serve application traffic over HTTPS.

---

## Design decisions

- All traffic should be over HTTPS
- All HTTP requests should be automatically redirected to HTTPS
- App teams should not be able to define routes over HTTP
- The platform team will manage the configuration of TLS termination for the application teams
- App teams continue to have the ability to self-service routes against the shared gateway

We decide to use [cert-manager](https://cert-manager.io/docs/) to generate and otherwise manage certificates.

---

## Deploy `cert-manager`

```shell
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
```

```shell
helm upgrade --install --create-namespace --namespace cert-manager \
  --set crds.enabled=true \
  --set "extraArgs={--enable-gateway-api}" \
  cert-manager jetstack/cert-manager
```

---

## Create a self-signed issuer

```yaml linenums="1"
--8<-- "https/selfsigned-issuer.yaml"
```

```shell
kubectl apply -f https/selfsigned-issuer.yaml
```

---

## Update the Gateway configuration

Review the below updated Gateway configuration:

```yaml linenums="1" hl_lines="8 15-17 22-25 36-39"
--8<-- "https/gateway-add-https.yaml"
```

Note the following:

- In addition to the HTTP listener on port 80, we add HTTPS listeners on port 443 for both `httpbin` and the `customers` hostnames.
- The HTTPS listeners are configured to terminate TLS and reference a certificate in Kubernetes secrets:  `httpbin-cert` for `httpbin` and `customers-cert` for the `customers` app.
- The `allowedRoutes` configuration that allows app teams to self-service their own routes is now specified at the HTTPS listener level.
- Routes can only be attached to the HTTP listener if they're defined in the _same_ namespace as the Gateway resource (`infra`).
- The annotation on line eight (8) enlists `cert-manager`'s certificate issuer to generate the certificates and store them in the specified secrets.

Apply the configuration:

```shell
kubectl apply -f https/gateway-add-https.yaml
```

---

## :white_check_mark: Test it

Access `httpbin` over TLS:

- A verbose HEAD request showing the TLS handshake:

    ```shell
    curl --insecure -v --head https://httpbin.example.com/ \
      --resolve httpbin.example.com:443:$GATEWAY_IP
    ```

- A call to the `/json` endpoint over https:

    ```shell
    curl --insecure https://httpbin.example.com/json \
      --resolve httpbin.example.com:443:$GATEWAY_IP
    ```

## Configure redirection

The platform team is the only party with the permission to apply this routing rule, to redirect HTTP requests to HTTPS with a 301 (redirected) response code:

```yaml linenums="1" hl_lines="12"
--8<-- "https/httpbin-to-https.yaml"
```

```shell
kubectl apply -f https/httpbin-to-https.yaml
```

To learn more about HTTPRoute rules and specifically filters, see the [relevant section in the Gateway API documentation](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.HTTPRouteRule).

---

## :white_check_mark: Test

Verify that you get a [301 response](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/301) when curling the http endpoint:

```shell
curl -v http://httpbin.example.com/ --resolve httpbin.example.com:80:$GATEWAY_IP
```

Here is a copy of the captured output:

```console linenums="1" hl_lines="12-13"
* Host httpbin.example.com:80 was resolved.
* IPv6: (none)
* IPv4: 34.121.222.176
*   Trying 34.121.222.176:80...
* Connected to httpbin.example.com (34.121.222.176) port 80
> GET / HTTP/1.1
> Host: httpbin.example.com
> User-Agent: curl/8.7.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 301 Moved Permanently
< location: https://httpbin.example.com:443/
< date: Tue, 07 May 2024 21:21:19 GMT
< content-length: 0
<
* Connection #0 to host httpbin.example.com left intact
```

# Summary

We now have an ingress configuration that functions over HTTPS using a shared gateway that accommodates the needs of both the platform team and multiple application teams.
