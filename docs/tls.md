# TLS

The objective is to configure the Gateway to serve `httpbin` over TLS.

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

```yaml linenums="1"
--8<-- "tls/selfsigned-issuer.yaml"
```

```shell
kubectl apply -f tls/selfsigned-issuer.yaml
```

---

## Add an HTTPS listener

Add an HTTPS listener for `httpbin.esuez.org` hostname on the gateway, configured to terminate TLS:

```yaml linenums="1" hl_lines="7 18-21"
--8<-- "tls/gateway-add-https.yaml"
```

```shell
kubectl apply -f tls/gateway-add-https.yaml
```

---

## :white_check_mark: Test it

[Access httpbin over TLS](https://httpbin.esuez.org/):


=== "Using DNS resolution"

    ```shell
    curl --insecure -v --head https://httpbin.esuez.org/
    ```

=== "Using `curl` name resolve flag"

    ```shell
    curl --insecure -v --head https://httpbin.esuez.org/ \
       --resolve httpbin.esuez.org:443:$GATEWAY_IP
    ```
