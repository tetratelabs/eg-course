# Monitor Envoy Gateway with Grafana

The following content is distilled from the [Envoy Gateway docs](https://gateway.envoyproxy.io/docs/tasks/observability/grafana-integration/).

The objective is to collect Gateway metrics with Prometheus and expose them through Grafana dashboards.

---

## Generate a load

```shell
while true; do
  curl --insecure --head https://httpbin.example.com/json --resolve httpbin.example.com:443:$GATEWAY_IP
  sleep 0.5
done
```

---

## Viewing proxy metrics directly with `egctl`

The `egctl` CLI provides an experimental `stats` subcommand for displaying metrics exposed by an Envoy proxy instance.

We used this command in a previous lab to watch the upstream retries metric (`envoy_cluster_upstream_rq_retry`).

Using labels to identify the proxy we wish to target, we can obtain its metrics with the following command:

```shell
egctl x stats envoy-proxy \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=infra \
  -n envoy-gateway-system
```

We can use the `--type=clusters` option to display metrics that are specific to Envoy clusters (upstream services) defined in its configuration:

```shell
egctl x stats envoy-proxy \
  -l gateway.envoyproxy.io/owning-gateway-name=eg \
  -l gateway.envoyproxy.io/owning-gateway-namespace=infra \
  -n envoy-gateway-system \
  --type clusters
```

---

## Deploy Observability tools

The Envoy Gateway project goes a step further, and provides a helm chart that makes it easy to ingest those metrics into Prometheus, and monitor our proxies through Grafana dashboards.

The following command will deploy all necessary observability tools to the `monitoring` namespace, including prometheus and grafana:

```shell
helm install eg-addons oci://docker.io/envoyproxy/gateway-addons-helm \
  --version v{{eg.version}} \
  --set opentelemetry-collector.enabled=true \
  -n monitoring --create-namespace
```

Confirm that Prometheus, Grafana, and other observability tools (loki, tempo, fluent-bit) are running in the `monitoring` namespace.

---

## Monitor Envoy

A LoadBalancer type service is already defined for Grafana in the `monitoring` namespace.

If you're running locally or don't have a public IP address associated with the service, you can use the `kubectl port-forward` command:

```shell
kubectl -n monitoring port-forward svc/grafana 3000:80
```

Visit [localhost:3000](http://localhost:3000) and login to grafana using `admin:admin`.

- The prometheus data source is already configured.
- Several Envoy monitoring dashboards have already be imported and can be seen in the `envoy-gateway` folder.

You will find four distinct dashboards:

- Envoy Global: monitor Envoy proxies (data plane)
- Envoy Gateway Global: monitor Envoy Gateway (control plane)
- Envoy Clusters: Envoy proxy metrics with cluster/service-level granularity (data plane)
- Resources Monitor: monitor resource utilization of Envoy proxies and Envoy Gateway

## Ad-hoc query metrics

A LoadBalancer type service is already defined for Prometheus in the `monitoring` namespace.

If you're running locally or don't have a public IP address associated with the service, you can use the `kubectl port-forward` command:

```shell
kubectl -n monitoring port-forward svc/prometheus 9090:80
```

Visit [localhost:9090](http://localhost:9090) and look for the retry metric from the [retries](retries.md/#review-the-proxys-stats) lab:

```promql
envoy_cluster_upstream_rq_retry{envoy_cluster_name="httproute/default/httpbin/rule/0"}
```
