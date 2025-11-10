# DDOT Dual Shipping Setup

This guide explains how to install and configure the Datadog Distribution of OpenTelemetry (DDOT) Collector using Helm, with dual shipping capabilities to send telemetry data to both Datadog and a secondary observability backend with native OTLP support.

## Overview

The DDOT Collector runs as a DaemonSet alongside the Datadog Agent, allowing you to collect OpenTelemetry telemetry from your applications and route it to multiple destinations.

## Prerequisites

- Kubernetes cluster
- Helm 3.x
- Datadog API key
- Secondary observability backend configuration (OTLP endpoint and authentication headers)

## Step 1: Enable DDOT Collector in Datadog Helm Chart

### 1.1 Configure datadog-values.yaml

Create or update your `datadog-values.yaml` file to enable the OpenTelemetry Collector:

```yaml
datadog:
  apiKey: <YOUR_DD_API_KEY>
  site: datadoghq.com  # Or your Datadog site (e.g., datadoghq.eu, us3.datadoghq.com, etc.)
  
  # Enable OpenTelemetry Collector
  otelCollector:
    enabled: true
    
    # Expose OTLP receiver ports
    ports:
      - containerPort: 4317
        name: otlp-grpc
        protocol: TCP
      - containerPort: 4318
        name: otlp-http
        protocol: TCP
```

### 1.2 Configure Your Application

Your application needs to be configured to send telemetry to the DDOT Collector running on the same node. Update your application's deployment manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-app
  labels:
    tags.datadoghq.com/env: "<ENV>"
    tags.datadoghq.com/service: "<SERVICE>"
    tags.datadoghq.com/version: "<VERSION>"
spec:
  template:
    metadata:
      labels:
        tags.datadoghq.com/env: "<ENV>"
        tags.datadoghq.com/service: "<SERVICE>"
        tags.datadoghq.com/version: "<VERSION>"
    spec:
      containers:
      - name: your-app
        image: your-app:latest
        env:
          # Get the host IP where DDOT Collector is running
          - name: HOST_IP
            valueFrom:
              fieldRef:
                fieldPath: status.hostIP
          
          # Kubernetes metadata for resource attributes
          - name: OTEL_K8S_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          
          - name: OTEL_K8S_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          
          - name: OTEL_K8S_POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          
          - name: OTEL_K8S_POD_ID
            valueFrom:
              fieldRef:
                fieldPath: metadata.uid
          
          # Configure OTLP endpoint to point to DDOT Collector
          - name: OTEL_EXPORTER_OTLP_ENDPOINT
            value: 'http://$(HOST_IP):4317'
          
          # Use gRPC protocol (or http/protobuf for port 4318)
          - name: OTEL_EXPORTER_OTLP_PROTOCOL
            value: 'grpc'
          
          # Set service name
          - name: OTEL_SERVICE_NAME
            value: "<SERVICE>"
          
          # Configure unified service tagging with extended Kubernetes resource attributes
          # These attributes are critical for the infraattributes processor to correctly
          # associate telemetry with infrastructure metadata
          - name: OTEL_RESOURCE_ATTRIBUTES
            value: >-
              service.name=$(OTEL_SERVICE_NAME),
              service.version=<VERSION>,
              k8s.namespace.name=$(OTEL_K8S_NAMESPACE),
              k8s.node.name=$(OTEL_K8S_NODE_NAME),
              k8s.pod.name=$(OTEL_K8S_POD_NAME),
              k8s.pod.uid=$(OTEL_K8S_POD_ID),
              k8s.container.name=your-app,
              host.name=$(OTEL_K8S_NODE_NAME),
              deployment.environment.name=$(OTEL_K8S_NAMESPACE)
```

**Key Application Configuration Points:**

- `HOST_IP`: Uses Kubernetes downward API to get the node's IP where the pod is running
- `OTEL_K8S_*`: Kubernetes metadata extracted via downward API (namespace, node, pod name/uid)
- `OTEL_EXPORTER_OTLP_ENDPOINT`: Points to the DDOT Collector on the same node (DaemonSet pattern)
- `OTEL_EXPORTER_OTLP_PROTOCOL`: Set to `grpc` for port 4317 or `http/protobuf` for port 4318
- `OTEL_SERVICE_NAME`: Your service name
- `OTEL_RESOURCE_ATTRIBUTES`: Extended resource attributes including:
  - Service identification (`service.name`, `service.version`)
  - Kubernetes context (`k8s.namespace.name`, `k8s.node.name`, `k8s.pod.name`, `k8s.pod.uid`, `k8s.container.name`)
  - Host and environment (`host.name`, `deployment.environment`)
  - These attributes enable the `infraattributes` processor to correlate telemetry with infrastructure tags

**Important**: The extended Kubernetes resource attributes (`k8s.pod.uid`, `k8s.node.name`, etc.) are critical for proper infrastructure correlation. Without these, you may experience [missing infrastructure tags](https://docs.datadoghq.com/opentelemetry/troubleshooting/?tab=datadogagentotlpingestion#infrastructure-tags-are-missing-from-telemetry) on your telemetry data.

## Step 2: Configure Dual Shipping to a Secondary Observability Backend

### 2.1 Create Custom OpenTelemetry Collector Configuration

To dual ship telemetry to both Datadog and a secondary observability backend with native OTLP support, you need to override the default DDOT configuration. Create an `otel-config.yaml` file:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  
  # Prometheus receiver for Collector health metrics
  prometheus:
    config:
      scrape_configs:
        - job_name: "otelcol"
          scrape_interval: 10s
          static_configs:
            - targets: ["0.0.0.0:8888"]

exporters:
  # Datadog exporter
  datadog:
    api:
      key: ${env:DD_API_KEY}
      site: ${env:DD_SITE}
  
  # Secondary backend OTLP HTTP exporter
  otlphttp:
    endpoint: https://your-backend.example.com/v1/traces
    headers:
      # Replace with your backend's authentication headers
      authorization: "Bearer <YOUR_API_TOKEN>"
      # Add any additional headers required by your backend
      # x-custom-header: "<VALUE>"
    compression: gzip

connectors:
  # Datadog connector for APM stats computation
  datadog/connector:
    traces:

processors:
  batch:
    timeout: 10s
  
  # Datadog Infrastructure Attributes processor
  # Enriches telemetry with infrastructure tags from the Datadog Agent
  infraattributes:
    cardinality: 2

service:
  pipelines:
    # Traces pipeline - dual ship to Datadog and OTLP backend
    traces:
      receivers: [otlp]
      processors: [batch, infraattributes]
      exporters: [datadog/connector, datadog, otlphttp]
    
    # Metrics pipeline - send to Datadog
    metrics:
      receivers: [otlp, datadog/connector, prometheus]
      processors: [batch, infraattributes]
      exporters: [datadog]
    
    # Logs pipeline - send to Datadog (add otlphttp to dual ship logs)
    logs:
      receivers: [otlp]
      processors: [batch, infraattributes]
      exporters: [datadog, otlphttp]
```

### 2.2 Update datadog-values.yaml with Custom Configuration

You can either include the configuration inline in your `datadog-values.yaml` or reference it as a separate file during Helm installation.

**Option A: Inline Configuration**

Update your `datadog-values.yaml`:

```yaml
datadog:
  apiKey: <YOUR_DD_API_KEY>
  site: datadoghq.com
  
  otelCollector:
    enabled: true
    
    ports:
      - containerPort: 4317
        name: otlp-grpc
        protocol: TCP
      - containerPort: 4318
        name: otlp-http
        protocol: TCP
    
    # Inline custom configuration
    config:
      receivers:
        otlp:
          protocols:
            grpc:
              endpoint: 0.0.0.0:4317
            http:
              endpoint: 0.0.0.0:4318
        
        prometheus:
          config:
            scrape_configs:
              - job_name: "otelcol"
                scrape_interval: 10s
                static_configs:
                  - targets: ["0.0.0.0:8888"]
      
      exporters:
        datadog:
          api:
            key: ${env:DD_API_KEY}
            site: ${env:DD_SITE}
        
        otlphttp:
          endpoint: https://your-backend.example.com/v1/traces
          headers:
            authorization: "Bearer <YOUR_API_TOKEN>"
          compression: gzip
      
      connectors:
        datadog/connector:
          traces:
      
      processors:
        batch:
          timeout: 10s
        
        infraattributes:
          cardinality: 2
      
      service:
        pipelines:
          traces:
            receivers: [otlp]
            processors: [batch, infraattributes]
            exporters: [datadog/connector, datadog, otlphttp]
          
          metrics:
            receivers: [otlp, datadog/connector, prometheus]
            processors: [batch, infraattributes]
            exporters: [datadog]
          
          logs:
            receivers: [otlp]
            processors: [batch, infraattributes]
            exporters: [datadog]
```

**Option B: External Configuration File**

Keep your custom configuration in a separate `otel-config.yaml` file and reference it during Helm installation (see deployment step below).

### 2.3 Deploy with Helm

Deploy the Datadog Agent with DDOT Collector:

**Using inline configuration:**

```bash
helm upgrade -i datadog datadog/datadog \
  -f datadog-values.yaml \
  --namespace datadog \
  --create-namespace
```

**Using external configuration file:**

```bash
helm upgrade -i datadog datadog/datadog \
  -f datadog-values.yaml \
  --set-file datadog.otelCollector.config=otel-config.yaml \
  --namespace datadog \
  --create-namespace
```

### 2.4 Verify Deployment

Check that the Datadog Agent and DDOT Collector are running:

```bash
# Check DaemonSet status
kubectl get daemonset -n datadog

# Check pod logs
kubectl logs -n datadog -l app=datadog -c otel-agent

# Verify OTLP endpoints are listening
kubectl port-forward -n datadog <pod-name> 4317:4317 4318:4318
```

## Configuration Notes

### Secondary Backend Integration

The configuration above uses the OTLP HTTP exporter to send telemetry to a secondary observability backend. To configure it for your specific backend:

- **Endpoint**: Set to your backend's OTLP endpoint (e.g., `https://your-backend.example.com/v1/traces`)
  - For traces: typically `/v1/traces`
  - For metrics: typically `/v1/metrics`
  - For logs: typically `/v1/logs`
  - Some backends use a single endpoint for all signals
- **Headers**: Configure authentication headers as required by your backend
  - Common patterns include `Authorization: Bearer <token>`, API keys, or custom headers
  - Consult your backend's documentation for specific requirements
- **Compression**: gzip is recommended for network efficiency and is widely supported

Replace the placeholder values with your actual backend credentials and endpoint information.

### Datadog Connector

The `datadog/connector` is essential for computing APM trace metrics. It must be:
1. Defined in the `connectors` section
2. Added to the traces pipeline exporters (before the datadog exporter)
3. Added to the metrics pipeline receivers

This ensures Datadog APM stats are properly calculated from your traces.

### Infrastructure Attributes Processor

The [`infraattributes` processor](https://github.com/DataDog/datadog-agent/tree/main/comp/otelcol/otlp/components/processor/infraattributesprocessor#readme) is a Datadog-specific processor that enriches telemetry with infrastructure tags from the Datadog Agent. It:

- Correlates telemetry data with infrastructure metadata (host tags, container tags, etc.)
- Uses resource attributes like `k8s.pod.uid`, `container.id`, or `process.pid` to match telemetry with infrastructure
- Adds Datadog-specific tags that enable infrastructure correlation in the UI
- Supports configurable cardinality levels (0=low, 1=orchestrator, 2=high)

**Configuration**:
```yaml
infraattributes:
  cardinality: 2  # Use level 2 for full tag enrichment
```

**Important**: For the processor to work correctly, your application must set the proper resource attributes (`k8s.pod.uid`, `k8s.node.name`, etc.) as shown in Step 1.2. See the [troubleshooting guide](https://docs.datadoghq.com/opentelemetry/troubleshooting/?tab=datadogagentotlpingestion#infrastructure-tags-are-missing-from-telemetry) if infrastructure tags are missing.

### Resource Detection

The `resourcedetection` processor automatically adds cloud provider and container metadata to your telemetry, enriching it with context about where it's running.

## Troubleshooting

### Application Can't Connect to DDOT Collector

1. Verify the DaemonSet is running on the same node as your application pod
2. Check that `HOST_IP` environment variable is correctly set
3. Verify firewall rules allow communication on ports 4317/4318

### Missing Infrastructure Tags on Telemetry

**Symptom**: Infrastructure tags (host, container, pod metadata) are missing from your traces, metrics, or logs in Datadog.

**Resolution**:

1. **Verify resource attributes**: Ensure your application is setting the required Kubernetes resource attributes:
   - `k8s.pod.uid` (most important for correlation)
   - `k8s.node.name`
   - `k8s.pod.name`
   - `k8s.namespace.name`
   - `container.id` (if available)

2. **Check infraattributes processor**: Confirm the processor is included in all pipelines (traces, metrics, logs)

3. **Verify Datadog Agent tagger**: The infraattributes processor relies on the Datadog Agent's tagger. Check Agent logs for tagger issues:
   ```bash
   kubectl logs -n datadog <agent-pod> -c agent
   ```

4. **Use debug exporter**: Add a debug exporter to verify attributes are present:
   ```yaml
   exporters:
     debug:
       verbosity: detailed
   
   service:
     pipelines:
       traces:
         exporters: [debug, datadog/connector, datadog, otlphttp]
   ```

For more details, see the [OpenTelemetry troubleshooting guide](https://docs.datadoghq.com/opentelemetry/troubleshooting/?tab=datadogagentotlpingestion#infrastructure-tags-are-missing-from-telemetry).

### No Data in Secondary Backend

1. Verify backend credentials and endpoint are correct
2. Check DDOT Collector logs for export errors
3. Ensure your network allows outbound HTTPS to your backend's endpoint
4. Verify the backend supports OTLP over HTTP
5. Check that authentication headers match your backend's requirements

### High Memory Usage

If you experience high memory usage, adjust the batch processor settings:

```yaml
processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
    send_batch_max_size: 2048
```

## References

- [Datadog DDOT Kubernetes DaemonSet Installation](https://docs.datadoghq.com/opentelemetry/setup/ddot_collector/install/kubernetes_daemonset?tab=helm)
- [Datadog Infrastructure Attributes Processor](https://github.com/DataDog/datadog-agent/tree/main/comp/otelcol/otlp/components/processor/infraattributesprocessor#readme)
- [OpenTelemetry Troubleshooting - Missing Infrastructure Tags](https://docs.datadoghq.com/opentelemetry/troubleshooting/?tab=datadogagentotlpingestion#infrastructure-tags-are-missing-from-telemetry)
- [OpenTelemetry Collector Configuration](https://opentelemetry.io/docs/collector/configuration/)
- [OTLP Exporter Configuration](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/otlphttpexporter)
- [Calendar Demo Application - Java with OpenTelemetry SDK](https://github.com/DataDog/opentelemetry-examples/tree/main/apps/rest-services/java/calendar)

