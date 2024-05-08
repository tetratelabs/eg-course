# Snippets

Capture the Gateway IP address for the gateway named `eg`:

```shell
export GATEWAY_IP=$(kubectl get gtw eg -o jsonpath='{.status.addresses[0].value}')
```
