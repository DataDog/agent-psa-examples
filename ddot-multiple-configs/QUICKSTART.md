# Quick Start Guide

This is a quick reference for getting started with DDOT Multiple Configs.

## Prerequisites

- Kubernetes cluster
- kubectl and helm installed
- Datadog API key and App key

## Installation

### 1. Create namespace and secret

```bash
kubectl create namespace ddot-demo

kubectl create secret generic datadog-secret \
  --from-literal api-key=<DD_API_KEY> \
  --from-literal app-key=<DD_APP_KEY> \
  --namespace ddot-demo
```

### 2. Install Datadog Operator

```bash
helm repo add datadog https://helm.datadoghq.com
helm repo update

helm install datadog-operator datadog/datadog-operator \
  --namespace ddot-demo
```

### 3. Deploy ConfigMap

```bash
kubectl apply -f k8s/configmaps/otel-config.yaml
```

This ConfigMap contains both the base configuration and the dual shipping override.

### 4. Update and deploy Datadog Agent

Update `datadog-agent.yaml`:
- Set `spec.global.site` to your Datadog site
- Set `spec.global.clusterName` to your cluster name

```bash
kubectl apply -f datadog-agent.yaml
```

### 5. Deploy sample application

```bash
kubectl apply -f k8s/calendar/deployment.yaml
kubectl apply -f k8s/calendar/service.yaml
```

## Verify in Datadog

1. **APM Traces**: https://app.datadoghq.com/apm/traces
2. **Infrastructure**: https://app.datadoghq.com/infrastructure
3. **Logs**: https://app.datadoghq.com/logs

Search for service: `calendar`

## View Logs

```bash
# List all pods
kubectl get pods -n ddot-demo

# Get Datadog Agent pod name (adjust based on actual pod names)
POD_NAME=$(kubectl get pods -n ddot-demo -o name | grep agent | head -1 | cut -d'/' -f2)

# View OTEL Collector logs
kubectl logs -n ddot-demo $POD_NAME -c otel-agent

# View Calendar logs
kubectl logs -n ddot-demo -l app.kubernetes.io/name=calendar
```

## Cleanup

To remove all resources:

```bash
# Delete sample application
kubectl delete -f k8s/calendar/

# Delete Datadog Agent
kubectl delete -f datadog-agent.yaml

# Delete ConfigMap
kubectl delete -f k8s/configmaps/otel-config.yaml

# Uninstall Datadog Operator
helm uninstall datadog-operator -n ddot-demo

# Delete secret
kubectl delete secret datadog-secret -n ddot-demo

# Delete namespace (optional)
kubectl delete namespace ddot-demo
```

## Common Issues

### Pods not starting

Check operator logs:
```bash
kubectl logs -n ddot-demo -l app.kubernetes.io/name=datadog-operator
```

### No data in Datadog

1. Verify API key is correct
2. Check Collector logs for errors
3. Verify outbound HTTPS is allowed

### ConfigMaps not loading

```bash
kubectl get configmap -n ddot-demo
kubectl describe configmap otel-configs -n ddot-demo
```

## Next Steps

- Read the full [README.md](README.md) for detailed documentation
- See [USAGE.md](USAGE.md) for advanced configuration patterns
- Check Datadog documentation for troubleshooting

