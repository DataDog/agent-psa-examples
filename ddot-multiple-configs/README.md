# DDOT Multiple Configs Setup with Datadog Operator

This guide demonstrates how to use multiple OpenTelemetry Collector configurations with the Datadog Distribution of OpenTelemetry (DDOT) Collector deployed via the Datadog Operator.

## Overview

The Datadog Operator supports loading multiple OpenTelemetry Collector configurations through Kubernetes ConfigMaps. Multiple configs are **merged together**, allowing you to:

- Start with a **base configuration** (similar to the [default included with the Operator](https://github.com/DataDog/datadog-operator/blob/main/internal/controller/datadogagent/feature/otelcollector/defaultconfig/defaultconfig.go))
- Add **override/extension configurations** for specific use cases
- Manage different concerns separately (base setup vs. custom extensions)

## Use Case: Base Config + Dual Shipping

This demo demonstrates a common pattern:

1. **Base configuration** (`otel-config-base.yaml`) - Standard DDOT setup with Datadog exporter
2. **Dual shipping configuration** (`otel-config-dualship.yaml`) - Adds an additional OTLP exporter

The configurations are **merged** by the OpenTelemetry Collector, resulting in traces/metrics/logs being sent to both Datadog and a secondary observability backend.

## Prerequisites

- Kubernetes cluster (1.20+)
- Helm 3.x
- kubectl configured to access your cluster
- Datadog API key and App key
- Datadog Operator (will be installed in the steps below)

## Installation Steps

### Step 1: Create Namespace

Create a dedicated namespace for the demo:

```bash
kubectl create namespace ddot-demo
```

### Step 2: Store API Keys as Kubernetes Secret

Create a secret containing your Datadog API and App keys:

```bash
kubectl create secret generic datadog-secret \
  --from-literal api-key=<YOUR_DD_API_KEY> \
  --from-literal app-key=<YOUR_DD_APP_KEY> \
  --namespace ddot-demo
```

Replace `<YOUR_DD_API_KEY>` and `<YOUR_DD_APP_KEY>` with your actual Datadog credentials.

### Step 3: Install the Datadog Operator

Add the Datadog Helm repository and install the Operator:

```bash
# Add Datadog Helm repository
helm repo add datadog https://helm.datadoghq.com
helm repo update

# Install the Datadog Operator
helm install datadog-operator datadog/datadog-operator \
  --namespace ddot-demo
```

Verify the Operator is running:

```bash
kubectl get pods -n ddot-demo
```

### Step 4: Deploy ConfigMap with Multiple OTEL Configurations

Apply the ConfigMap that contains both base and dual shipping configurations:

```bash
kubectl apply -f k8s/configmaps/otel-config.yaml
```

This single ConfigMap contains two configuration files that will be merged by the OpenTelemetry Collector:
- `otel-config-base.yaml` - Standard DDOT setup
- `otel-config-dualship.yaml` - Adds OTLP exporter for dual shipping

Verify ConfigMap is created:

```bash
kubectl get configmap otel-configs -n ddot-demo -o yaml
```

You should see both configuration keys in the ConfigMap data.

### Step 5: Update datadog-agent.yaml

Before deploying the Datadog Agent, update the `datadog-agent.yaml` file:

1. Set your Datadog site (e.g., `datadoghq.com`, `us5.datadoghq.com`, `datadoghq.eu`)
2. Set your cluster name
3. Review the namespace (should be `ddot-demo`)

```yaml
spec:
  global:
    site: datadoghq.com  # Update this
    clusterName: my-cluster  # Update this
```

### Step 6: Deploy the Datadog Agent

Deploy the DatadogAgent custom resource:

```bash
kubectl apply -f datadog-agent.yaml
```

Wait for the Agent DaemonSet to be ready:

```bash
kubectl get daemonset -n ddot-demo -w
```

Check the Agent pods are running:

```bash
# List all pods in the namespace
kubectl get pods -n ddot-demo

# You should see pods created by the Operator (names will vary based on your configuration)
```

### Step 7: Deploy Sample Application

Deploy the Calendar sample application:

```bash
kubectl apply -f k8s/calendar/deployment.yaml
kubectl apply -f k8s/calendar/service.yaml
```

Verify the application is running:

```bash
kubectl get pods -n ddot-demo -l app.kubernetes.io/name=calendar
```


## Configuration Details

### Multiple ConfigMaps Pattern

The `datadog-agent.yaml` references a ConfigMap with multiple configuration files that are **merged** by the OpenTelemetry Collector:

```yaml
features:
  otelCollector:
    enabled: true
    conf:
      configMap:
        name: otel-configs
        items:
          - key: otel-config-base.yaml      # Base configuration
            path: otel-config-base.yaml
          - key: otel-config-dualship.yaml  # Dual shipping extension
            path: otel-config-dualship.yaml
```

### How Configuration Merging Works

When multiple configuration files are provided:

1. **Components (receivers, exporters, processors, connectors)** are merged by name
   - Different names → Both included
   - Same name → Later definition overrides

2. **Pipelines** are merged by name
   - Different pipeline names → Both included
   - Same pipeline name → Later definition overrides (use this for extending exporters)

### Base Configuration (otel-config-base.yaml)

**Purpose**: Standard DDOT setup similar to the [default Operator configuration](https://github.com/DataDog/datadog-operator/blob/main/internal/controller/datadogagent/feature/otelcollector/defaultconfig/defaultconfig.go).

**Key components**:
- **OTLP Receiver**: Accepts telemetry via gRPC (4317) and HTTP (4318)
- **Prometheus Receiver**: Collects Collector health metrics
- **Infraattributes Processor**: Enriches telemetry with infrastructure tags
- **Batch Processor**: Batches data for efficient export
- **Memory Limiter**: Prevents OOM
- **Datadog Connector**: Computes APM statistics from traces
- **Datadog Exporter**: Sends all telemetry to Datadog

**Pipelines**:
- **traces**: OTLP → processors → [Datadog Connector + Datadog Exporter]
- **metrics**: [OTLP + Prometheus + Connector] → processors → Datadog Exporter
- **logs**: OTLP → processors → Datadog Exporter

### Dual Shipping Configuration (otel-config-dualship.yaml)

**Purpose**: Extends the base configuration to add dual shipping capability.

**What it adds**:
- **OTLP HTTP Exporter**: New exporter for secondary backend
- **Updated Pipelines**: Overrides pipelines to include both exporters

**Result after merge**:
- All base components remain
- Additional OTLP exporter is added
- Pipelines now export to both Datadog AND secondary backend

**Example pipeline override**:
```yaml
# In dualship config - this REPLACES the traces pipeline from base
service:
  pipelines:
    traces:
      receivers: [otlp]  # Same receivers
      processors: [memory_limiter, batch, infraattributes]  # Same processors
      exporters: [datadog/connector, datadog, otlphttp]  # + otlphttp added
```

## Verification

### Check Collector Health

View the Collector container logs:

```bash
# First, list all pods to see what's running
kubectl get pods -n ddot-demo

# Get a Datadog Agent pod name (adjust the grep pattern based on actual pod names)
POD_NAME=$(kubectl get pods -n ddot-demo -o name | grep agent | head -1 | cut -d'/' -f2)

# Or if you know the exact pod name from the list above
# POD_NAME=<pod-name-from-list>

# Check OTEL Collector logs
kubectl logs -n ddot-demo $POD_NAME -c otel-agent
```

You should see the Collector startup logs showing all three configurations loaded.

### Verify in Datadog UI

1. **Traces**: Navigate to APM → Traces in Datadog
   - You should see traces from the `calendar` service
   - Check for proper tagging (env, service, version)

2. **Metrics**: Navigate to Metrics Explorer
   - Search for `otelcol_*` metrics (Collector health metrics)
   - Search for application metrics

3. **Logs**: Navigate to Logs Explorer
   - Filter by `service:calendar`
   - Verify logs have infrastructure tags

4. **Infrastructure**: Navigate to Infrastructure → Host Map
   - Your Kubernetes nodes should appear
   - Traces and metrics should be correlated with infrastructure

### Check OTLP Endpoints

Verify OTLP endpoints are accessible:

```bash
# Port-forward to a Datadog Agent pod
kubectl port-forward -n ddot-demo $POD_NAME 4317:4317 4318:4318

# In another terminal, test connectivity
grpcurl -plaintext localhost:4317 list
```

## Advanced Configuration

### Modifying Configurations

To modify a configuration:

1. Edit the ConfigMap file:
   ```bash
   vim k8s/configmaps/otel-config.yaml
   ```

2. Update either the base config or dual shipping config key as needed

3. Apply the changes:
   ```bash
   kubectl apply -f k8s/configmaps/otel-config.yaml
   ```

4. Restart the Datadog Agent pods to pick up changes:
   ```bash
   # First, find the daemonset name
   kubectl get daemonsets -n ddot-demo
   
   # Restart using the actual daemonset name from above
   kubectl rollout restart daemonset/<daemonset-name> -n ddot-demo
   
   # Or delete all pods in the namespace to force restart
   kubectl delete pods --all -n ddot-demo
   ```

### Adding More Configurations

To add additional configuration overrides:

1. Edit `k8s/configmaps/otel-config.yaml`
2. Add a new key under `data:` with your additional configuration:
   ```yaml
   data:
     otel-config-base.yaml: |
       # existing base config...
     otel-config-dualship.yaml: |
       # existing dualship config...
     otel-config-custom.yaml: |
       # your new custom config...
   ```
3. Apply the ConfigMap:
   ```bash
   kubectl apply -f k8s/configmaps/otel-config.yaml
   ```
4. Update `datadog-agent.yaml` to reference the new key:
   ```yaml
   items:
     - key: otel-config-base.yaml
       path: otel-config-base.yaml
     - key: otel-config-dualship.yaml
       path: otel-config-dualship.yaml
     - key: otel-config-custom.yaml
       path: otel-config-custom.yaml
   ```
5. Reapply the DatadogAgent resource:
   ```bash
   kubectl apply -f datadog-agent.yaml
   ```

### Infrastructure Attributes Processor

The `infraattributes` processor enriches telemetry with Datadog-specific infrastructure tags. It's critical for correlating telemetry with infrastructure in the Datadog UI.

**Configuration**:
```yaml
processors:
  infraattributes:
    cardinality: 2  # 0=low, 1=orchestrator, 2=high (recommended)
```

**Required resource attributes** (set by your application):
- `k8s.pod.uid` - Most important for pod correlation
- `k8s.node.name` - Node identification
- `k8s.pod.name` - Pod name
- `k8s.namespace.name` - Namespace
- `k8s.container.name` - Container name

See the Calendar deployment for proper configuration examples.

## Application Configuration

Your application must send telemetry to the DDOT Collector on the same node. The Calendar deployment shows the proper configuration:

**Key environment variables**:

```yaml
env:
  # Get host IP where DDOT runs
  - name: HOST_IP
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
  
  # Configure OTLP endpoint
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: 'http://$(HOST_IP):4317'
  
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: 'grpc'
  
  # Kubernetes metadata for infraattributes processor
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: >-
      service.name=$(OTEL_SERVICE_NAME),
      k8s.pod.uid=$(OTEL_K8S_POD_UID),
      k8s.node.name=$(OTEL_K8S_NODE_NAME),
      ...
```

See `k8s/calendar/deployment.yaml` for the complete configuration.

## Troubleshooting

### Pods Not Starting

Check the Operator logs:
```bash
kubectl logs -n ddot-demo -l app.kubernetes.io/name=datadog-operator
```

Check the Agent pod events:
```bash
kubectl describe pod -n ddot-demo $POD_NAME
```

### ConfigMaps Not Loading

Verify ConfigMaps exist:
```bash
kubectl get configmap -n ddot-demo
```

Check ConfigMap contents:
```bash
kubectl get configmap otel-config-traces -n ddot-demo -o yaml
```

### No Data in Datadog

1. **Check API key**: Verify your secret is correct
   ```bash
   kubectl get secret datadog-secret -n ddot-demo -o yaml
   ```

2. **Check Collector logs**: Look for export errors
   ```bash
   kubectl logs -n ddot-demo $POD_NAME -c otel-agent | grep -i error
   ```

3. **Verify application is sending data**: Check application logs
   ```bash
   kubectl logs -n ddot-demo -l app.kubernetes.io/name=calendar
   ```

4. **Test connectivity**: Ensure outbound HTTPS is allowed to Datadog

### Missing Infrastructure Tags

If infrastructure tags are missing from your telemetry:

1. **Verify resource attributes**: Check your application sets required K8s attributes
2. **Check infraattributes processor**: Ensure it's in all pipelines
3. **View Agent tagger logs**: Look for tagger issues in Agent container logs

See the [Datadog troubleshooting guide](https://docs.datadoghq.com/opentelemetry/troubleshooting/?tab=datadogagentotlpingestion#infrastructure-tags-are-missing-from-telemetry) for more details.

### High Memory Usage

Adjust batch processor settings in your ConfigMaps:

```yaml
processors:
  batch:
    timeout: 10s
    send_batch_size: 1024      # Adjust based on your needs
    send_batch_max_size: 2048   # Adjust based on your needs
```

Add memory limiter processor:

```yaml
processors:
  memory_limiter:
    check_interval: 5s
    limit_percentage: 80
    spike_limit_percentage: 25
```

## Cleanup

To remove all resources:

```bash
# Delete application
kubectl delete -f k8s/calendar/

# Delete Datadog Agent
kubectl delete -f datadog-agent.yaml

# Delete ConfigMaps
kubectl delete -f k8s/configmaps/

# Uninstall Operator
helm uninstall datadog-operator -n ddot-demo

# Delete namespace
kubectl delete namespace ddot-demo
```

## References

- [Datadog Operator Installation Guide](https://docs.datadoghq.com/opentelemetry/setup/ddot_collector/install/kubernetes_daemonset?tab=datadogoperator)
- [Datadog Operator PR #1980 - Multiple Configs Support](https://github.com/DataDog/datadog-operator/pull/1980)
- [Datadog Infrastructure Attributes Processor](https://github.com/DataDog/datadog-agent/tree/main/comp/otelcol/otlp/components/processor/infraattributesprocessor)
- [OpenTelemetry Collector Configuration](https://opentelemetry.io/docs/collector/configuration/)
- [Datadog OpenTelemetry Troubleshooting](https://docs.datadoghq.com/opentelemetry/troubleshooting/)
- [Calendar Demo Application](https://github.com/DataDog/opentelemetry-examples/tree/main/apps/rest-services/java/calendar)

## Support

For issues or questions:
- Datadog Support: https://docs.datadoghq.com/help/
- OpenTelemetry Community: https://opentelemetry.io/community/
- Datadog Operator Issues: https://github.com/DataDog/datadog-operator/issues

