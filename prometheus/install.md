# Install Prometheus on Minikube using Helm

We will install Prometheus in your existing Minikube cluster using the `kube-prometheus-stack` Helm chart. This is a practical DevOps setup because it can also install Grafana, Alertmanager, and Prometheus Operator.

Configuration

- Kubernetes context: `minikube`
- Namespace: `monitoring`
- Installation method: Helm
- Helm chart: `prometheus-community/kube-prometheus-stack`

## Step 1: Verify your Kubernetes context

Run these commands in your terminal (PowerShell, Git Bash, or a terminal with `kubectl` configured):

```
kubectl config current-context
```

Ensure the current context is `minikube`. If it is not, switch to it:

```
kubectl config use-context minikube
```

## Step 2: Create the monitoring namespace

```
kubectl create namespace monitoring
```

If the namespace already exists, verify it instead:

```
kubectl get namespace monitoring
```

## Step 3: Add the Prometheus Community Helm repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Update the repository index:

```bash
helm repo update
```

## Step 4: Install Prometheus using Helm

For a local Minikube environment, install the stack with reduced resource requests and persistent storage disabled for a simpler lab setup.

```bash
helm install prometheus prometheus-community/prometheus --namespace monitoring
```

Output:

```bash
NAME: prometheus
LAST DEPLOYED: Mon Oct 19 19:00:47 2026
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.monitoring.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9090

Prometheus alertmanager can be accessed via port 9093 on the following DNS name from within your cluster:
prometheus-alertmanager.monitoring.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=alertmanager,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9093

Prometheus Pushgateway can be accessed via port 9091 on the following DNS name from within your cluster:
prometheus-prometheus-pushgateway.monitoring.svc.cluster.local


Get the Pushgateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=prometheus-pushgateway,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/
```


## Step 5: Verify the installation

Check the Helm release:

```
helm list -n monitoring
helm status prometheus -n monitoring
```

Check all monitoring resources:

```
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get pvc -n monitoring
```

Wait for the pods to become ready:

```
kubectl wait --for=condition=Ready pods \
  --all -n monitoring --timeout=300s
```

If a pod remains in `Pending` or `CrashLoopBackOff`, inspect it:

```
kubectl describe pods -n monitoring
kubectl get events -n monitoring --sort-by=.lastTimestamp
```

## Step 6: Access the Prometheus UI

Use port-forwarding, which is usually the simplest option for a local Minikube cluster.

First, find the Prometheus service:

```
kubectl get svc -n monitoring
```

The service is typically named `prometheus-kube-prometheus-prometheus`. Forward its web port:

```
kubectl port-forward -n monitoring \
  svc/prometheus-kube-prometheus-prometheus 9090:9090
```

Open this URL in your browser:

[http://localhost:9090](http://localhost:9090?utm_source=chatgpt.com)

Keep the terminal running while you use the UI.

## Step 7: Verify metric collection

In the Prometheus UI, open Status → Targets or enter the following query in the expression bar:

```
up
```

You should see metrics for successfully scraped targets. You can also test:

```
kube_pod_info
```

This query returns Kubernetes pod information collected through the monitoring stack.
