# Multiple Configs Usage Guide

This document explains how to use multiple OpenTelemetry Collector configurations with the Datadog Operator and provides patterns for common use cases.

## Understanding Multiple Configs

The Datadog Operator supports loading multiple OpenTelemetry Collector configuration files, which are then **merged** by the Collector itself. This enables a powerful pattern:

1. **Base Configuration** - Standard DDOT setup (similar to the [default Operator config](https://github.com/DataDog/datadog-operator/blob/main/internal/controller/datadogagent/feature/otelcollector/defaultconfig/defaultconfig.go))
2. **Override/Extension Configurations** - Additional configs that extend or modify the base

### Why Use Multiple Configs?

- **Separation of Concerns**: Keep base setup separate from custom extensions
- **Easy Dual Shipping**: Add secondary backends without modifying base config
- **Maintainability**: Update standard configs without touching customizations
- **Flexibility**: Enable/disable features by adding/removing config files
- **Version Control**: Track base and overrides separately

## How Configuration Merging Works

When you provide multiple configuration files, the OpenTelemetry Collector merges them using these rules:

### 1. Component Merging (receivers, exporters, processors, connectors)

Components are **merged by name**:

- **Different names** → Both included in final config
- **Same name** → Later definition completely replaces earlier one (no deep merge)

**Example**:

```yaml
# Config key 1: otel-config-base.yaml
exporters:
  datadog:
    api:
      key: ${env:DD_API_KEY}
      site: ${env:DD_SITE}

# Config key 2: otel-config-dualship.yaml
exporters:
  otlphttp:  # Different name → Added to config
    endpoint: https://backend.example.com

# Result: Both exporters available
exporters:
  datadog: ...
  otlphttp: ...
```

### 2. Pipeline Merging

Pipelines are **merged by pipeline name**:

- **Different pipeline names** → Both included
- **Same pipeline name** → Later definition **completely replaces** earlier one

**This is key for extending pipelines!**

**Example**:

```yaml
# Config key 1: otel-config-base.yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [datadog]

# Config key 2: otel-config-dualship.yaml
service:
  pipelines:
    traces:  # Same name → Replaces entire pipeline
      receivers: [otlp]
      processors: [batch]
      exporters: [datadog, otlphttp]  # Now has both exporters!

# Result: Pipeline with both exporters
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [datadog, otlphttp]
```

### 3. Service-Level Configuration

Service-level settings (telemetry, extensions) from the last file take precedence.

## Pattern 1: Base + Dual Shipping (This Repo)

This is the primary pattern demonstrated in this repository.

### Structure

```yaml
# ConfigMap with multiple keys
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-configs
data:
  otel-config-base.yaml: |
    # Standard DDOT configuration
    receivers:
      otlp: ...
      prometheus: ...
    
    exporters:
      datadog: ...
    
    processors:
      batch: ...
      infraattributes: ...
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [datadog]
  
  otel-config-dualship.yaml: |
    # Add OTLP exporter and override pipelines
    exporters:
      otlphttp: ...  # New exporter
    
    service:
      pipelines:
        traces:  # Override
          receivers: [otlp]
          exporters: [datadog, otlphttp]  # Both!
```

### DatadogAgent Configuration

```yaml
features:
  otelCollector:
    enabled: true
    conf:
      configMap:
        name: otel-configs
        items:
          - key: otel-config-base.yaml      # Loaded first
            path: otel-config-base.yaml
          - key: otel-config-dualship.yaml  # Merged second
            path: otel-config-dualship.yaml
```

### Result

- All base components (receivers, processors, connectors) remain
- OTLP exporter is added
- Pipelines export to **both** Datadog and secondary backend

### Use Cases

- Dual shipping to Datadog + Grafana/New Relic/Elastic
- Testing new backends without disrupting production (Datadog)
- Compliance requirements (data to multiple systems)
- Migration scenarios

## Pattern 2: Base + Feature Extensions

Extend base configuration with additional features.

### Example: Adding a Custom Receiver

```yaml
# Config key: otel-config-base.yaml
receivers:
  otlp: ...

exporters:
  datadog: ...

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [datadog]

---
# Config key: otel-config-custom.yaml
receivers:
  jaeger:  # Add Jaeger receiver
    protocols:
      grpc:
        endpoint: 0.0.0.0:14250

service:
  pipelines:
    traces:  # Override to include new receiver
      receivers: [otlp, jaeger]
      exporters: [datadog]
```

### Use Cases

- Adding support for legacy instrumentation (Jaeger, Zipkin)
- Enabling specialized receivers (Kafka, AWS X-Ray)
- Custom metric collection

## Pattern 3: Environment-Specific Overrides

Use different overrides for different environments.

### Example: Production vs. Staging

```yaml
# Config key: otel-config-base.yaml
# Standard config for all environments

---
# Config key: otel-config-prod.yaml
processors:
  memory_limiter:
    limit_percentage: 80  # Stricter limits in prod

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [datadog]

---
# Config key: otel-config-staging.yaml
exporters:
  debug:  # Debug logging in staging
    verbosity: detailed

processors:
  memory_limiter:
    limit_percentage: 90  # More lenient in staging

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug, datadog]  # Include debug in staging
```

### Deployment

Deploy different DatadogAgent resources per environment, each referencing appropriate configs.

## Pattern 4: Feature Flags

Enable/disable features by adding/removing config files.

### Example: Sampling Configuration

```yaml
# Config key: otel-config-base.yaml
# Standard config

---
# Config key: otel-config-sampling.yaml (optional)
processors:
  probabilistic_sampler:
    sampling_percentage: 10

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [probabilistic_sampler, batch]  # Add sampling
      exporters: [datadog]
```

To enable sampling: Add `otel-config-sampling.yaml` to the items list.
To disable sampling: Remove it from the items list.

## Best Practices

### 1. Keep Base Config Clean

Your base config should contain:
- Standard receivers (OTLP, Prometheus)
- Essential processors (batch, infraattributes, memory_limiter)
- Datadog connector and exporter
- Basic pipelines

### 2. Use Overrides for Extensions

Put these in override configs:
- Additional exporters
- Custom processors
- New receivers
- Modified pipelines

### 3. Document Your Configs

Add comments explaining:
- Why the override exists
- What it adds or changes
- Configuration dependencies

### 4. Test Incrementally

When adding overrides:
1. Test base config alone first
2. Add one override at a time
3. Verify each addition works
4. Check merged result in Collector logs

### 5. Order Matters

The order of `items` in the DatadogAgent config matters:
- List base configs first
- Then add overrides
- Last override wins for conflicting settings

## Troubleshooting

### Config Not Loading

**Symptoms**: Changes not reflected, Collector not starting

**Debug steps**:

1. **Check ConfigMap exists**:
   ```bash
   kubectl get configmap otel-configs -n ddot-demo
   ```

2. **Verify ConfigMap content**:
   ```bash
   kubectl get configmap otel-configs -n ddot-demo -o yaml
   ```

3. **Check for YAML syntax errors**:
   ```bash
   kubectl get configmap otel-configs -n ddot-demo -o json | jq -r '.data."otel-config-base.yaml"' | yq eval
   ```

4. **Review Collector logs**:
   ```bash
   kubectl logs -n ddot-demo <agent-pod> -c otel-agent | grep -i config
   ```

### Unexpected Merge Results

**Symptoms**: Pipeline not working as expected, components missing

**Debug steps**:

1. **Enable debug logging** in your config:
   ```yaml
   service:
     telemetry:
       logs:
         level: debug
   ```

2. **Check effective configuration**:
   Look for "final config" in Collector startup logs

3. **Verify component names**:
   Ensure no accidental name conflicts

4. **Check pipeline definitions**:
   Remember: later pipeline definitions replace earlier ones

### Pipeline Override Not Working

**Problem**: Added exporter in override but it's not being used

**Solution**: Make sure you're overriding the entire pipeline, not just adding the exporter:

```yaml
# Wrong - just adding exporter (won't work)
exporters:
  otlphttp: ...

# Correct - override entire pipeline
exporters:
  otlphttp: ...

service:
  pipelines:
    traces:
      receivers: [otlp]  # Must list all
      processors: [batch, infraattributes]  # Must list all
      exporters: [datadog, otlphttp]  # Now includes both
```

## Advanced Topics

### Conditional Configs

Use different configs based on conditions (requires external tooling like Kustomize):

```yaml
# kustomization.yaml
bases:
  - base/

patchesStrategicMerge:
  - overlays/production.yaml  # Or staging.yaml, dev.yaml
```

### Config Templating

Use Helm or Kustomize to template ConfigMap values:

```yaml
# values.yaml
dualShipEndpoint: "https://backend.example.com"
dualShipToken: "secret-token"

# ConfigMap template
exporters:
  otlphttp:
    endpoint: {{ .Values.dualShipEndpoint }}
    headers:
      authorization: "Bearer {{ .Values.dualShipToken }}"
```

### Validation

Validate your OTEL configs before applying:

```bash
# Use otelcol validate (if you have the binary)
# Extract the config from ConfigMap first
kubectl get configmap otel-configs -n ddot-demo -o json | jq -r '.data."otel-config-base.yaml"' > /tmp/otel-config-base.yaml
otelcol validate --config=/tmp/otel-config-base.yaml
```

## Additional Resources

- [Datadog Operator Documentation](https://docs.datadoghq.com/containers/datadog_operator/)
- [Datadog Operator PR #1980](https://github.com/DataDog/datadog-operator/pull/1980) - Multiple configs support
- [OpenTelemetry Collector Configuration](https://opentelemetry.io/docs/collector/configuration/)
- [DDOT Documentation](https://docs.datadoghq.com/opentelemetry/setup/ddot_collector/)
