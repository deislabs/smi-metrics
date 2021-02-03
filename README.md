# SMI Metrics API

This repository is an implementation of the [Traffic Metrics Spec](https://github.com/servicemeshinterface/smi-spec/blob/master/traffic-metrics.md) which follows the format of the official [Metrics API](https://github.com/kubernetes-incubator/metrics-server) being built on top of kubernetes to get metrics about pods and nodes.
Here, the metrics are about the inbound and outbound requests i.e their Golden Metrics like p99_latency, p55_latency, success_count, etc for a particular workload.

## Installation

For Linkerd:

```bash
helm template chart --set adapter=linkerd | kubectl apply -f -
```

For Istio

```bash
helm template chart --set adapter=istio | kubectl apply -f -
```

## Roadmap

The API supports [linkerd](https://linkerd.io/)  and [Istio](https://istio.io/) right now.
The support for  [Consul](https://learn.hashicorp.com/consul/) is being worked up on right now and the API and responses will have the same structure unless there are no changes to the spec.

## Working

The SMI metrics api is a Kubernetes [APIService](https://kubernetes.io/docs/tasks/access-kubernetes-api/setup-extension-api-server/) as seen in the [installation manifest](https://github.com/servicemeshinterface/smi-metrics/blob/master/chart/templates/apiservice.yaml#L5),
which is a way of extending the Kubernetes API.

We will perform installation of the SMI Metrics API w.r.t linkerd. Make sure linkerd is installed and is running as per the instructions [here](https://linkerd.io/2/getting-started/), This API can be installed by running the following command

```bash
helm repo add smi https://servicemeshinterface.github.io/smi-metrics
helm install smi-metrics smi/smi-metrics --set adapter=linkerd
```

The installation of the APIService can be verified by running

```bash
kubectl get apiservice | grep smi
v1alpha1.metrics.smi-spec.io           default/dev-smi-metrics   True        25h
v1alpha1.metrics.smi-spec.io           default/dev-smi-metrics   True        25h
```

The APIService first informs the kubernetes API about itself and the resource types that it exposes, which can be viewed by running

```bash
kubectl api-resources | grep smi
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
daemonsets                                     metrics.smi-spec.io            true         TrafficMetrics
deployments                                    metrics.smi-spec.io            true         TrafficMetrics
namespaces                                     metrics.smi-spec.io            false        TrafficMetrics
pods                                           metrics.smi-spec.io            true         TrafficMetrics
statefulsets                                   metrics.smi-spec.io            true         TrafficMetrics
```

This means that, the APIService can be queried regarding the above mentioned resource types.

Now that the SMI APIService is installed, metric queries can be done through the kubernetes API.

Because metrics only return the last 30s by default, if you'd like to see the metrics, make sure there is something happening on your cluster. The easiest way to do this is to run `linkerd dashboard` and see the traffic generated by viewing that page.

```bash
kubectl get --raw /apis/metrics.smi-spec.io/v1alpha1/namespaces/linkerd/deployments/linkerd-web | jq
```

Output:

```json
{
  "kind": "TrafficMetrics",
  "apiVersion": "metrics.smi-spec.io/v1alpha1",
  "metadata": {
    "name": "linkerd-web",
    "namespace": "linkerd",
    "selfLink": "/apis/metrics.smi-spec.io/v1alpha1/namespaces/linkerd/deployments/linkerd-web",
    "creationTimestamp": "2019-06-17T14:26:22Z"
  },
  "timestamp": "2019-06-17T14:26:22Z",
  "window": "30s",
  "resource": {
    "kind": "Deployment",
    "namespace": "linkerd",
    "name": "linkerd-web"
  },
  "edge": {
    "direction": "from",
    "resource": null
  },
  "metrics": [
    {
      "name": "p99_response_latency",
      "unit": "ms",
      "value": "296875m"
    },
    {
      "name": "p90_response_latency",
      "unit": "ms",
      "value": "268750m"
    },
    {
      "name": "p50_response_latency",
      "unit": "ms",
      "value": "162500m"
    },
    {
      "name": "success_count",
      "value": "73492m"
    },
    {
      "name": "failure_count",
      "value": "0"
    }
  ]
}
```

As we can see, the golden metrics of a particular resource can be retrieved by querying the API with a path format `/apis/metrics.smi-spec.io/v1alpha1/namespaces/{Namespace}/{Kind}/{ResourceName}`

Queries for golden metrics on edges i.e paths associated with a particular resource, for example linkerd-controller can also be done by adding `/edges` to the path.

```bash
kubectl get --raw /apis/metrics.smi-spec.io/v1alpha1/namespaces/linkerd/deployments/linkerd-controller/edges | jq
```

Output:

```json
{
  "kind": "TrafficMetricsList",
  "apiVersion": "metrics.smi-spec.io/v1alpha1",
  "metadata": {
    "selfLink": "/apis/metrics.smi-spec.io/v1alpha1/namespaces/linkerd/deployments/linkerd-controller/edges"
  },
  "resource": {
    "kind": "Deployment",
    "namespace": "linkerd",
    "name": "linkerd-controller"
  },
  "items": [
    {
      "kind": "TrafficMetrics",
      "apiVersion": "metrics.smi-spec.io/v1alpha1",
      "metadata": {
        "name": "linkerd-controller",
        "namespace": "linkerd",
        "selfLink": "/apis/metrics.smi-spec.io/v1alpha1/namespaces/linkerd/deployments/linkerd-controller/edges",
        "creationTimestamp": "2019-06-17T14:51:57Z"
      },
      "timestamp": "2019-06-17T14:51:57Z",
      "window": "30s",
      "resource": {
        "kind": "Deployment",
        "namespace": "linkerd",
        "name": "linkerd-controller"
      },
      "edge": {
        "direction": "from",
        "resource": {
          "kind": "Deployment",
          "namespace": "linkerd",
          "name": "linkerd-web"
        }
      },
      "metrics": [
        {
          "name": "p99_response_latency",
          "unit": "ms",
          "value": "294"
        },
        {
          "name": "p90_response_latency",
          "unit": "ms",
          "value": "240"
        },
        {
          "name": "p50_response_latency",
          "unit": "ms",
          "value": "150"
        },
        {
          "name": "success_count",
          "value": "28580m"
        },
        {
          "name": "failure_count",
          "value": "0"
        }
      ]
    },
    {
      "kind": "TrafficMetrics",
      "apiVersion": "metrics.smi-spec.io/v1alpha1",
      "metadata": {
        "name": "linkerd-controller",
        "namespace": "linkerd",
        "selfLink": "/apis/metrics.smi-spec.io/v1alpha1/namespaces/linkerd/deployments/linkerd-controller/edges",
        "creationTimestamp": "2019-06-17T14:51:58Z"
      },
      "timestamp": "2019-06-17T14:51:57Z",
      "window": "30s",
      "resource": {
        "kind": "Deployment",
        "namespace": "linkerd",
        "name": "linkerd-controller"
      },
      "edge": {
        "direction": "to",
        "resource": {
          "kind": "Deployment",
          "namespace": "linkerd",
          "name": "linkerd-prometheus"
        }
      },
      "metrics": [
        {
          "name": "p99_response_latency",
          "unit": "ms",
          "value": "368"
        },
        {
          "name": "p90_response_latency",
          "unit": "ms",
          "value": "247199m"
        },
        {
          "name": "p50_response_latency",
          "unit": "ms",
          "value": "120731m"
        },
        {
          "name": "success_count",
          "value": "1008100m"
        },
        {
          "name": "failure_count",
          "value": "0"
        }
      ]
    }
  ]
}
```

## Development

- Get [tilt](https://tilt.dev/), run `tilt up`.
- The prometheus API client has been mocked out with a tool `mockery`. When
  bumping the API client version, a new mock will need to be generated. This can
  be done by checking out the correct version of the API client repo, running
  `mockery -name API` and copying the `mocks` folder into `pkg/metrics/mocks`.

## Admin

- `/debug`
- `/metrics`
- `/status`

## TODO

### API Questions

- ObjectMeta includes OwnerReferences, Labels and Annotations. Should any of
  these be included as part of TrafficMetrics?
- ObjectReference has ResourceVersion and APIVersion, pull these in?

### Internal details

- export prometheus for client-go
- integrate swagger with apiservice (OpenAPI AggregationController)

## Contributing

Please refer to [CONTRIBUTING.md](./CONTRIBUTING.md) for more information on contributing.
