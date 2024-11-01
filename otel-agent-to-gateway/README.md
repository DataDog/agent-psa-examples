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

4. Install Datadog Agent with embedded OTel collector

```shell
helm upgrade -i agent-hippo datadog/datadog \
  --namespace calendar-hippo \
  --values datadog-values.yaml \
  --set-file datadog.otelCollector.config=collector-config.yaml \
  --set agents.nodeSelector."alpha\\.eksctl\\.io/nodegroup-name"=mng-calendar-hippo
```

Generate manifest:
```shell
helm template agent-hippo datadog/datadog \
  --namespace calendar-hippo \
  --values datadog-values.yaml \
  --set-file datadog.otelCollector.config=collector-config.yaml \
  --set agents.nodeSelector."alpha\\.eksctl\\.io/nodegroup-name"=mng-calendar-hippo \
  --output-dir ./k8s
```

5. Go to app.datadoghq.com and verify telemetry data.

6. Build BYOC collector

Download BYOC manifest file
```shell
curl -o manifest.yaml https://raw.githubusercontent.com/DataDog/datadog-agent/main/comp/otelcol/collector-contrib/impl/manifest.yaml
```

Download BYOC Docker file
```shell
curl -o Dockerfile.byoc https://raw.githubusercontent.com/DataDog/datadog-agent/main/Dockerfiles/agent-ot/Dockerfile.agent-otel
```

Build BYOC image
```shell
docker build --file Dockerfile.byoc --tag agent-byoc:0.104.0 .
```

Publish BYOC image to the registry:
```shell
docker tag agent-byoc:0.104.0 601427279990.dkr.ecr.us-east-2.amazonaws.com/krlv/agent-byoc:0.104.0
docker push 601427279990.dkr.ecr.us-east-2.amazonaws.com/krlv/agent-byoc:0.104.0
```

7. Use BYOC Agent image

```yaml
# datadog-values.yaml
agents:
  image:
    repository: 601427279990.dkr.ecr.us-east-2.amazonaws.com/krlv/agent-byoc
    tag: 0.104.0
```

8. Connect Agent with OTel collector to `collector-contrib` deployed as a gateway

```yaml
# collecotr-config.yaml
...
exporters:
  ...
  loadbalancing:
    protocol:
      otlp:
        tls:
          insecure: true
    resolver:
      dns:
        hostname: collector-gw-opentelemetry-collector.demo-zapus.svc.cluster.local

  pipelines:
    traces:
      ...
      exporters: [debug, datadog, datadog/connector, loadbalancing]
    logs:
      ...
      exporters: [debug, datadog, loadbalancing]

```
