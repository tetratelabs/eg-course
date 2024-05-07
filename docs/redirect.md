# Redirect HTTP to HTTPS

---

## Configure redirection

```yaml linenums="1"
--8<-- "redirect/httpbin-redirect-https.yaml"
```

```shell
kubectl apply -f httpbin-redirect-https.yaml
```

---

## :white_check_mark: Test

Verify that you get a [301 response](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/301) when curling the http endpoint:

```shell
curl -v http://httpbin.esuez.org/
```

TODO - consider capturing the output
