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

### 3. Install Datadog Agent

Install the agent via Helm:

```shell
helm upgrade -i otel-agent datadog/datadog \
  --values datadog-values.yaml \
  --set-file datadog.otelCollector.config=collector-config.yaml \
  --namespace demo
```

### 4. Install demo app

We will use [busybox](https://hub.docker.com/_/busybox) container with a `while true; do echo $(date); sleep 1; done` loop
as a demo app. To install it run:

```shell
kubectl apply -f ./logger/k8s/deployment.yaml
```

### 5. Add `filelog` receiver to OpenTelemetry Collector config

1. Define `filelog` receiver in the OTel collector config as follows:

```yaml
# collector-config.yaml
receivers:
  ...
  filelog:
    exclude: []
    include:
      - /var/log/pods/*/*/*.log
    include_file_name: false
    include_file_path: true
    operators:
      - id: container-parser
        max_log_size: 102400
        type: container
    retry_on_failure:
      enabled: true
    start_at: end
...
service:
  telemetry:
    logs:
      level: debug
  pipelines:
    ...
    logs:
      receivers: [filelog, otlp]
      processors: [memory_limiter, infraattributes, batch]
      exporters: [debug, datadog]
```

2. Enable logs collection in OTel Collector:

```yaml
# datadog-values.yaml
datadog:
  ...
  otelCollector:
    enabled: true
    logs:
      enabled: true
    ...
```

Check [Logs Explorer](https://app.datadoghq.com/logs) to confirm containers logs are delivered and indexed:

![Container logs in Logs Explorer](./assets/2025-01-23_23-49-16.png)

### 6. (Optional) Exclude `otel-agent` container logs

To prevent logs looping, exclude logs from the collector's containers:

```yaml
# collector-config.yaml
receivers:
  ...
  filelog:
    # exclude logs from all containers named otel-agent
    exclude:
      - /var/log/pods/*/otel-agent/*.log
    include:
      - /var/log/pods/*/*/*.log
    include_file_name: false
    include_file_path: true
    operators:
      - id: container-parser
        max_log_size: 102400
        type: container
    retry_on_failure:
      enabled: true
    start_at: end
```

## Enable logs collection in the Datadog Agent with Embedded OpenTelemetry Collector

### 7. Enable logs collection in the Datadog Agent

```yaml
# datadog-values.with-logs.yaml
datadog:
  ...
  logs:
    enabled: true
    containerCollectAll: true
    containerCollectUsingFiles: true
    autoMultiLineDetection: false
  containerExcludeLogs: "name:.*"
  containerIncludeLogs: "name:ddlogger"
```

> [!NOTE]
> Inclusion takes precedence over exclusion. For example, to only monitor `checkout` or `payment` containers, 
> first exclude all other container names and then specify which containers to include:
> ```yaml
>   containerExcludeLogs: "name:.*"
>   containerIncludeLogs: "name:checkout name:payment"
> ```

