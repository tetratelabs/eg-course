# HTTPS

The objective of this scenario is to serve our applications over HTTPS.

---

## Deploy `cert-manager`

We decide to use [cert-manager](https://cert-manager.io/docs/) to generate and otherwise manage certificates.

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

## Add an HTTPS listener

Add an HTTPS listener for `httpbin.example.com` hostname on the gateway, configured to terminate TLS:

```yaml linenums="1" hl_lines="7 14-16 21-24 35-38"
--8<-- "https/gateway-add-https.yaml"
```

```shell
kubectl apply -f https/gateway-add-https.yaml
```

---

## :white_check_mark: Test it

Access `httpbin` over TLS:

```shell
curl --insecure -v --head https://httpbin.example.com/ \
    --resolve httpbin.example.com:443:$GATEWAY_IP
```

## Configure redirection

```yaml linenums="1" hl_lines="11 28"
--8<-- "https/httpbin-to-https.yaml"
```

```shell
kubectl apply -f https/httpbin-to-https.yaml
```

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
