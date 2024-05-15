# Observability

The following content is distilled from the [Envoy Gateway docs](https://gateway.envoyproxy.io/v1.0.1/tasks/observability/grafana-integration/).

The objective is to collect Gateway metrics with Prometheus and expose them through Grafana dashboards.

---

## Generate a load

```shell
while true; do
  curl --head https://httpbin.esuez.org/json --resolve httpbin.esuez.org:443:$GATEWAY_IP
  sleep 0.5
done
```

---

## Deploy Prometheus

```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Then:

```shell
helm upgrade --install prometheus prometheus-community/prometheus -n monitoring --create-namespace
```

Expose the prometheus dashboard:

```shell
kubectl -n monitoring port-forward deploy prometheus-server 9090
```

## Query metrics

Visit [localhost:9090](http://localhost:9090) and look for the retry metric from the [retries](retries.md/#review-the-proxys-stats) lab:

```promql
envoy_cluster_upstream_rq_retry{envoy_cluster_name="httproute/default/httpbin/rule/0"}
```

---

## Install Grafana

```shell
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Then:

```shell
helm upgrade --install grafana grafana/grafana -f https://raw.githubusercontent.com/envoyproxy/gateway/latest/examples/grafana/helm-values.yaml -n monitoring --create-namespace
```

Expose the Grafana dashboard:

```shell
kubectl -n monitoring port-forward svc/grafana 8080
```

## Configure Grafana

Visit [localhost:8080](http://localhost:8080) and login to grafana using `admin:admin`.

Configure the prometheus data source, using the service URL:

```
http://prometheus-server.monitoring.svc.cluster.local/
```

## Import dashboards

- [Envoy Global](https://raw.githubusercontent.com/envoyproxy/gateway/main/examples/grafana/dashboards/envoy-global.json)

- [Envoy Clusters](https://raw.githubusercontent.com/envoyproxy/gateway/main/examples/grafana/dashboards/envoy-clusters.json)

- [Envoy Pod Resources](https://raw.githubusercontent.com/envoyproxy/gateway/main/examples/grafana/dashboards/envoy-pod-resource.json)

## Visit the dashboards

Monitor your gateways in style.