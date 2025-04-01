# Collect Kubernetes logs and application logs written to stdout/stderr with Filelog Receiver.

The [Filelog Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver) tails and parses logs from files.

## Prerequisites

1. Kubernetes cluster
2. [Helm](https://helm.sh/docs/intro/install/)

## Install Datadog Agent with Embedded OpenTelemetry Collector

### 1. Install/update helm

Install [Datadog](https://github.com/DataDog/Helm-charts/)'s helm chart.

```shell
helm repo add datadog https://helm.datadoghq.com
helm repo update
```

### 2. Store API key as a Kubernetes secret

Create a secret that contains your Datadog API key. This secret is used in the manifest to deploy the Datadog Agent.

```shell
kubectl create secret generic datadog-secret \
  --from-literal api-key=<DD_API_KEY> \
  --from-literal app-key=<DD_APP_KEY> \
  --namespace demo
```

### 3. Install the Datadog Operator

To deploy the Datadog Agent with the Operator using a minimum number of steps, use 
the [`datadog-operator` Helm chart](https://github.com/DataDog/helm-charts/tree/main/charts/datadog-operator).

#### 3.1 Install the Datadog Operator via Helm:

```shell
helm upgrade -i otel-agent datadog/datadog-operator --namespace demo
```

#### 3.2 Configure `datadog-agent.yaml`

Create a file, datadog-agent.yaml, that contains the spec of your `DatadogAgent` deployment configuration:

```yaml
apiVersion: datadoghq.com/v2alpha1
kind: DatadogAgent
metadata:
  name: datadog
spec:
  global:
    credentials:
      apiSecret:
        secretName: datadog-secret
        keyName: api-key
      appSecret:
        secretName: datadog-secret
        keyName: app-key

  override:
    # Node Agent configuration
    nodeAgent:
      image:
        name: "public.ecr.aws/datadog/agent:7.64.1-ot-beta"
        pullPolicy: Always

  # Enable Features
  features:
    apm:
      enabled: true
    orchestratorExplorer:
      enabled: true
    processDiscovery:
      enabled: true
    liveProcessCollection:
      enabled: true
    usm:
      enabled: true
    clusterChecks:
      enabled: true
    otelCollector:
      enabled: true
      ports:
        - containerPort: 4317
          hostPort: 4317
          name: otel-grpc
        - containerPort: 4318
          hostPort: 4318
          name: otel-http
      conf:
        configData: |-
          receivers:
            prometheus:
              config:
                scrape_configs:
                  - job_name: "datadog-agent"
                    scrape_interval: 10s
                    static_configs:
                      - targets:
                          - 0.0.0.0:8888
            otlp:
              protocols:
                grpc:
                  endpoint: 0.0.0.0:4317
                http:
                  endpoint: 0.0.0.0:4318
          exporters:
            debug:
              verbosity: detailed
            datadog:
              api:
                key: ${env:DD_API_KEY}
                site: ${env:DD_SITE}
          processors:
            infraattributes:
              cardinality: 2
            batch:
              timeout: 10s
            memory_limiter:
              check_interval: 5s
              limit_percentage: 80
              spike_limit_percentage: 25
          connectors:
            datadog/connector:
              traces:
          service:
            pipelines:
              traces:
                receivers: [otlp]
                processors: [memory_limiter, infraattributes, batch]
                exporters: [debug, datadog, datadog/connector]
              metrics:
                receivers: [otlp, datadog/connector, prometheus]
                processors: [memory_limiter, infraattributes, batch]
                exporters: [debug, datadog]
              logs:
                receivers: [otlp]
                processors: [memory_limiter, infraattributes, batch]
                exporters: [debug, datadog]
```

#### 3.3 Deploy the Datadog Agent

```shell
kubectl apply -f ./datadog-agent.yaml
```
