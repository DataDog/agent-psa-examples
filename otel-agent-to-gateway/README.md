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

3. Deploy Demo app:
To install:
```shell
helm upgrade -i calendar-hippo ./deploys/calendar/ \
  --namespace calendar-hippo \
  --set image.repository=datadog/opentelemetry-examples \
  --set image.tag=calendar-java-20240916 \
  --set replicaCount=2 \
  --set resources.requests.cpu=200m \
  --set resources.requests.memory=500M \
  --set nodeSelector."alpha\\.eksctl\\.io/nodegroup-name"=mng-calendar-hippo
```

Generate manifest:
```shell
helm template calendar-hippo ./deploys/calendar/ \
  --namespace calendar-hippo \
  --set image.repository=datadog/opentelemetry-examples \
  --set image.tag=calendar-java-20240916 \
  --set replicaCount=2 \
  --set resources.requests.cpu=200m \
  --set resources.requests.memory=500M \
  --set nodeSelector."alpha\\.eksctl\\.io/nodegroup-name"=mng-calendar-hippo \
  --output-dir ../../../../../agent-psa-examples/otel-agent-to-gateway/k8s
```

4. 

