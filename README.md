# K8s Monitoring and Redis Cluster Setup

## Prerequisites
Ensure you have a Linux system with basic dependencies installed.

## Step 1: Install MicroK8s with Storage & MetalLB

```bash
mkdir ~/project
cd ~/project
```

Run the following script to install MicroK8s with necessary addons:

```bash
curl -fsSL https://raw.githubusercontent.com/prayag-sangode/config-scripts/refs/heads/main/kubernetes/microk8s-install/scripts/microk8s-ub.sh | bash
```

## Step 2: Install Prometheus and Grafana

```bash
mkdir monitoring
cd monitoring
```

Install kube-prometheus-stack using the following commands:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version 68.3.2 -n monitoring --create-namespace
```

Check installed Helm releases:

```bash
helm -n monitoring ls
```

Expose Prometheus and Grafana services:

```bash
kubectl patch svc kube-prometheus-stack-prometheus -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc kube-prometheus-stack-grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
```

Or use port-forwarding:

```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090 --address 0.0.0.0
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80 --address 0.0.0.0
```

## Step 3: Install Redis Cluster with Monitoring

```bash
cd ~/project
mkdir redis-cluster
cd redis-cluster
```

Create a values file for Redis cluster:

```yaml
cat > redis-cluster-values.yaml <<EOF
master:
  persistence:
    enabled: false

replica:
  persistence:
    enabled: false

metrics:
  enabled: true

  serviceMonitor:
    enabled: true
    port: http-metrics
    namespace: "monitoring"
    interval: 30s
    scrapeTimeout: "10s"
    honorLabels: true
    additionalLabels:
      release: kube-prometheus-stack
EOF
```

Install Redis cluster using Helm:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm upgrade --install redis-cluster bitnami/redis --version 20.6.3 \
  -f redis-cluster-values.yaml -n redis-cluster --create-namespace
```

### Grafana Dashboard for Redis
Use **Grafana Dashboard ID**: `11835` to monitor Redis metrics.

## Step 4: Test Redis Cluster Connectivity

Deploy an Nginx pod for testing:

```bash
kubectl create deployment nginx-test-deployment --image=nginx
kubectl expose deployment nginx-test-deployment --port=80 --target-port=80 \
  --name=nginx-test-service --type=ClusterIP
```

Access the pod shell:

```bash
kubectl exec -it pod/$(kubectl get pods -l app=nginx-test-deployment -o jsonpath="{.items[0].metadata.name}") -- /bin/bash
```

Install Redis CLI:

```bash
apt-get update && apt-get install -y redis-tools
```

Connect to Redis:

```bash
redis-cli -h redis-cluster-master.redis-cluster.svc.cluster.local -p 6379
```

Authenticate and configure max memory:

```bash
AUTH <password>
CONFIG SET maxmemory 512mb
CONFIG GET maxmemory
```

Benchmark Redis performance:

```bash
redis-benchmark -h redis-cluster-master.redis-cluster.svc.cluster.local -p 6379 -a t5faetMBCc -n 100000 -c 50
