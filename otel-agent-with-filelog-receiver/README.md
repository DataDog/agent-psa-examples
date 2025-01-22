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
  --namespace staging
```

### 3. Install Datadog Agent

Install the agent via Helm:

```shell
helm upgrade -i otel-agent datadog/datadog \
  --values datadog-values.yaml \
  --set-file datadog.otelCollector.config=collector-config.yaml \
  --namespace staging
```

### 4. Install demo app

We will use [busybox](https://hub.docker.com/_/busybox) container with a `while true; do echo $(date); sleep 1; done` loop
as a demo app. To install it run:

```shell
kubectl apply -f ./logger/k8s/deployment.yaml
```

