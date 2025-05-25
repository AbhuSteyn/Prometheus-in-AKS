
# Prometheus on Azure AKS Interview Guide

## Overview
Prometheus is an open-source monitoring and alerting toolkit popular for containerized environments. When deployed on Azure AKS, Prometheus collects metrics from various services, providing real-time insights into performance and health. This guide covers architecture details, deployment with ServiceMonitor, scaling techniques, securing access to dashboards, common queries, and alert configuration.

---

## 1. Why Prometheus on AKS?

**Benefits & Use Cases:**
- **Centralized Metrics Collection:**  
  Aggregates metrics from all nodes, pods, and services running on AKS.
- **Real-Time Monitoring & Alerting:**  
  Enables immediate detection of anomalies and performance bottlenecks.
- **Scalability:**  
  Designed to handle massive volumes of metrics data with sharding or federation.
- **Cost Effectiveness:**  
  Open-source and well-integrated with the Kubernetes ecosystem.

**Real-World Example:**  
Imagine a microservices-based production environment where performance issues may be scattered across many pods. Prometheus collects CPU, memory, and custom application metrics from every service, allowing you to pinpoint issues and trigger alerts before downtime occurs.

---

## 2. Prometheus Architecture on AKS

**Core Components:**
- **Prometheus Server:**  
  Scrapes targets (nodes, pods) at configured intervals and stores metrics in a time-series database.
- **Prometheus Operator:**  
  Simplifies deployment and management of Prometheus on Kubernetes. It introduces custom resources such as `Prometheus`, `ServiceMonitor`, and `Alertmanager`.
- **ServiceMonitor:**  
  A custom resource used by the Prometheus Operator to define how services should be scraped. It specifies selectors for services, endpoint settings, and scrape intervals.
- **Alertmanager:**  
  Manages alerts sent from Prometheus, including grouping, silencing, and routing notifications.
- **Grafana (Optional):**  
  Often used in combination with Prometheus for rich visualizations of metrics.

---

## 3. Detailed Setup Steps

### Step 1: Provision an AKS Cluster

Using the Azure CLI:
```bash
# Log in to Azure
az login

# Create a resource group
az group create --name prometheus-rg --location eastus

# Create an AKS cluster with monitoring enabled
az aks create --resource-group prometheus-rg --name PrometheusCluster --node-count 3 --enable-addons monitoring --generate-ssh-keys
```

### Step 2: Install the Prometheus Operator Using Helm

First, add the Helm repository and install the operator:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus Operator (kube-prometheus-stack) in the desired namespace
kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

### Step 3: Deploy a Sample Service and ServiceMonitor

**a. Deploy a Sample Application (e.g., a simple Nginx deployment)**
```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:latest
        name: nginx
        ports:
        - containerPort: 80
```
Apply the deployment:
```bash
kubectl apply -f nginx-deployment.yaml -n default
```

**b. Create a Service exposing the sample application**
```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```
Apply the service:
```bash
kubectl apply -f nginx-service.yaml -n default
```

**c. Create a ServiceMonitor to instruct Prometheus to scrape the sample service**
```yaml
# nginx-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-servicemonitor
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: nginx
  namespaceSelector:
    matchNames:
    - default
  endpoints:
  - port: 80
    interval: 15s
```
Apply the ServiceMonitor:
```bash
kubectl apply -f nginx-servicemonitor.yaml -n monitoring
```

### Step 4: Scale and Secure the Prometheus Dashboard

**Scaling Prometheus:**
- **Horizontal Pod Autoscaling:**  
  Configure resource requests/limits for Prometheus components and use Kubernetes HPA if needed.
- **Data Federation:**  
  Use federation to aggregate data from multiple Prometheus servers if required.

**Securing Dashboard Access:**
1. **Ingress Configuration:**  
   Create an Ingress that exposes the Prometheus and Grafana dashboards.  
2. **Authentication & TLS:**  
   Use basic authentication, OAuth2 proxy, or Azure AD integration to secure dashboard endpoints.
3. **RBAC:**  
   Enforce Kubernetes RBAC to restrict dashboard and Prometheus API access.

*Example Ingress snippet for Grafana (similar applies for Prometheus dashboard):*
```yaml
# grafana-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/auth-type: "basic"
    nginx.ingress.kubernetes.io/auth-secret: "grafana-basic-auth"
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
spec:
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-grafana
            port:
              number: 80
  tls:
  - hosts:
    - grafana.example.com
    secretName: grafana-tls
```
Generate a secret for basic auth and create TLS certificates as needed.

---

## 4. Common Prometheus Queries

Here are some example PromQL queries you might use:
- **CPU Usage:**  
  ```promql
  sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
  ```
- **Memory Usage:**  
  ```promql
  sum(container_memory_usage_bytes) by (pod)
  ```
- **HTTP Request Rate:**  
  ```promql
  rate(http_requests_total[1m])
  ```
- **Pod Availability Status:**  
  ```promql
  kube_pod_status_phase{phase="Running"}
  ```
These queries help monitor resource consumption and service performance in a live AKS environment.

---

## 5. Alerting in Prometheus

**Creating Alert Rules:**
Alert rules are defined in YAML files and loaded into Prometheus by the Prometheus Operator.  
*Example alert rule file (`cpu-alerts.yaml`):*
```yaml
groups:
- name: cpu_alerts
  rules:
  - alert: HighCPUUsage
    expr: sum(rate(container_cpu_usage_seconds_total[2m])) by (pod) > 0.8
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage detected on pod {{ $labels.pod }}"
      description: "CPU usage is above 80% for more than 2 minutes."
```
Apply the alert rule (depends on your Prometheus operator configuration).

**Alertmanager Integration:**
Prometheus sends alerts to Alertmanager, which handles routing, grouping, and notifications (email, Slack, etc.). Ensure Alertmanager is configured with a `config.yaml` that defines these routes.

---

## 6. Genuine Interview Questions & Detailed Answers

### Q1: Why use Prometheus on AKS?
**Detailed Answer:**  
"Prometheus provides a robust solution for collecting and analyzing metrics in dynamic, containerized environments. On AKS, it enables real-time monitoring, scales with the cluster, and integrates with Kubernetes via ServiceMonitors. This centralization simplifies troubleshooting and optimizes resource management."

### Q2: How does the ServiceMonitor work in Prometheus?
**Detailed Answer:**  
"A ServiceMonitor is a custom resource defined by the Prometheus Operator. It tells Prometheus which Kubernetes services to scrape and what endpoints to target. By labeling your services correctly and defining scrape intervals, you can automatically integrate new services into your monitoring solution without modifying Prometheus configuration."

### Q3: How do you scale Prometheus for large environments?
**Detailed Answer:**  
"Scaling involves a few strategies: 
- **Horizontal Scaling:** Adjusting pod replicas and using a federation model to aggregate data from multiple servers.
- **Resource Tuning:** Ensuring sufficient CPU and memory resources, and potentially using persistent volumes for long-term storage.
- **Federation:** In very large deployments, federating multiple Prometheus instances helps distribute the load."

### Q4: What strategies would you use to secure the Prometheus dashboard?
**Detailed Answer:**  
"Securing the dashboard involves:
- **Ingress Controls:** Exposing the dashboard through an Ingress with TLS encryption.
- **Authentication:** Implementing basic authentication or integrating with OAuth2/Azure AD.
- **RBAC:** Configuring Kubernetes RBAC policies to limit access to the Prometheus namespace and endpoints."

### Q5: What are some common Prometheus queries and their use cases?
**Detailed Answer:**  
"For example, a query to monitor CPU utilization is `sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)`, which helps identify pods with high CPU consumption. Similarly, querying `sum(container_memory_usage_bytes) by (pod)` assists in tracking memory usage trends across your cluster."

---

## Conclusion

Deploying Prometheus on Azure AKS provides a scalable, cost-effective monitoring solution for containerized environments. This guide details the setup—including ServiceMonitors, scaling approaches, securing dashboard access, common PromQL queries, and alert configurations—along with sample interview questions and detailed answers. Mastering these topics will help you confidently discuss your monitoring strategies during technical interviews.

