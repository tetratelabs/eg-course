# Traffic Splitting

This scenario demonstrates traffic splitting:  how the `weight` field in an HttpRoute can be used to specify the distribution of traffic between two backend references, two services.

## Context

The `customers` service should already be deployed in the default namespace (see [Shared Gateway](shared-gw.md)).

## Instructions

1. Deploy `customers-v2` backing deployment:

    ```yaml linenums="1"
    --8<-- "traffic-split/customers-v2.yaml"
    ```

    ```shell
    kubectl apply -f traffic-split/customers-v2.yaml
    ```

1. Define services for each v1 and v2 subsets:

    ```yaml linenums="1"
    --8<-- "traffic-split/customers-subsets.yaml"
    ```

    ```shell
    kubectl apply -f traffic-split/customers-subsets.yaml
    ```

1. Define an HttpRoute that splits traffic 80/20 between v1 and v2:

    ```yaml linenums="1"
    --8<-- "traffic-split/customers-route.yaml"
    ```

    ```shell
    kubectl apply -f traffic-split/customers-route.yaml
    ```

## Test it

Send a number of test request to the `customers` route:

```shell
for i in {1..10}; do
  curl http://customers.esuez.org/ --resolve customers.esuez.org:80:$GATEWAY_IP
done
```

About 80% of requests should go to v1.

Responses from the `v2` service can be distinguished from `v1` by their payload:  v2 returns customer names and cities, whereas v1 returns only customer names.
