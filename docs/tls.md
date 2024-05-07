# TLS

The objective here is to serve `httpbin` over TLS.

---

## Deploy `cert-manager`

We decide to let [cert-manager](https://cert-manager.io/docs/) manage certificates on our behalf.

```shell
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
```

```shell
helm upgrade --install --create-namespace --namespace cert-manager \
  --set installCRDs=true \
  --set featureGates=ExperimentalGatewayAPISupport=true \
  cert-manager jetstack/cert-manager
```

---

## Create a self-signed issuer

```shell
kubectl apply -f selfsigned-issuer.yaml
```

```yaml linenums="1"
--8<-- "tls/selfsigned-issuer.yaml"
```

---

## Add an HTTPS listener

Add an HTTPS listener for `httpbin.esuez.org` hostname on the gateway, configured to terminate TLS:

```shell
kubectl apply -f gateway-add-https.yaml
```

```yaml linenums="1"
--8<-- "tls/gateway-add-https.yaml"
```

---

## :white_check_mark: Test it

```shell
curl --insecure -v --head https://httpbin.esuez.org/
```
