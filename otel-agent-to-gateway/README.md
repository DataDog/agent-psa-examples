# Connect the Agent with embedded OTel collector to the OTel gateway

## Prerequisites

1. Kubernetes cluster
2. Helm

## Steps

1. Install/update helm
2. Deploy OTel collector as a gateway 

```shell
helm template collector-gw open-telemetry/opentelemetry-collector \
  --namespace demo-zapus \
  --values otel-values.yaml \
  --set nodeSelector."alpha\\.eksctl\\.io/nodegroup-name"=mng-demo-zapus \
  --output-dir ./k8s
```


