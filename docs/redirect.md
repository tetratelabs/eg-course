# Redirect HTTP to HTTPS

---

## Configure redirection

```yaml linenums="1" hl_lines="11 28"
--8<-- "redirect/httpbin-to-https.yaml"
```

```shell
kubectl apply -f redirect/httpbin-to-https.yaml
```

---

## :white_check_mark: Test

Verify that you get a [301 response](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/301) when curling the http endpoint:

```shell
curl -v http://httpbin.esuez.org/
```

Here is a copy of the captured output:

```console linenums="1" hl_lines="12-13"
* Host httpbin.esuez.org:80 was resolved.
* IPv6: (none)
* IPv4: 34.121.222.176
*   Trying 34.121.222.176:80...
* Connected to httpbin.esuez.org (34.121.222.176) port 80
> GET / HTTP/1.1
> Host: httpbin.esuez.org
> User-Agent: curl/8.7.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 301 Moved Permanently
< location: https://httpbin.esuez.org:443/
< date: Tue, 07 May 2024 21:21:19 GMT
< content-length: 0
<
* Connection #0 to host httpbin.esuez.org left intact
```
