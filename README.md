- In This Project we are Setting Up Prometheus Operator via OLM to Monitor a Node with 3 Nginx Services

## Step 1 : Install Operator Lifecycle Manager (OLM), 
a tool to help manage the Operators running on your cluster.

- OLM is the package manager for Kubernetes Operators.
- It handles installation, upgrades, and dependency management of Operators.
- Ensures consistent Operator lifecycle management across clusters.

```bash
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.32.0/install.sh | bash -s v0.32.0
```
It Will Deploys:

    olm-operator (manages Operator installations)

    catalog-operator (manages Operator catalogs)

    packageserver (serves Operator metadata)
- # Verify installation
```bash
kubectl get pods -n olm
```

## Step 2:Install the operator by running the following command:
This Operator will be installed in the "operators" namespace and will be usable from all namespaces in the cluster.


```bash
kubectl create -f https://operatorhub.io/install/prometheus.yaml
```

- After install, watch your operator come up using next command.

```bash
kubectl get csv -n operators
# Wait For Few Second and Run This Command Result:
controlplane ~ ➜  kubectl get csv -n operators
NAME                         DISPLAY               VERSION   REPLACES                     PHASE
prometheusoperator.v0.70.0   Prometheus Operator   0.70.0    prometheusoperator.v0.65.1   Pending

```
## Step 3: Create NameSpace
```bash
kubectl create namespace monitoring  # Isolate monitoring resources
```

## Step 4: Define OperatorGroup (limits where Operator can deploy resources)
```bash
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: monitoring-operatorgroup
  namespace: monitoring
spec:
  targetNamespaces:
  - monitoring  # Only watch this namespace
```
### Step 5:Subscribe to Prometheus Operator
```bash
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: prometheus
  namespace: monitoring
spec:
  channel: beta          # Update channel
  name: prometheus       # Operator name in OperatorHub
  source: operatorhubio-catalog  # Catalog source
  sourceNamespace: olm   # Where catalogs are installed
  installPlanApproval: Automatic  # Auto-updates
```
What Happens?
- OLM installs the Operator and its CRDs (Prometheus, ServiceMonitor, etc.).

- The Operator deploys a controller that reconciles Prometheus resources.

The Subscription resource is a crucial component when installing Operators through OLM (Operator Lifecycle Manager). Let me break down why we need it and what each field means:

Why Do We Need a Subscription?
A Subscription:

- Tells OLM which Operator to install and from where.

- Manages automatic updates of the Operator.

- Specifies the update channel (e.g., stable, beta).

- Links to the CatalogSource (where OLM finds Operators).

Without a Subscription, OLM wouldn’t know:

- Which Operator to deploy

- Which version to install

- How to handle updates

## step 6:Deploy Prometheus Instance
Why?
- The Operator doesn’t deploy Prometheus by default—we must define a Prometheus CR.

- This CR configures Prometheus settings (scraping, storage, etc.).
```bash
nano prometheus-instance.yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: main
  namespace: monitoring
spec:
  serviceAccountName: prometheus  # RBAC for scraping
  serviceMonitorSelector: {}       # Monitor all ServiceMonitors
  podMonitorSelector: {}           # Monitor all PodMonitors
  resources:
    requests:
      memory: 400Mi   # Avoid OOM kills
  securityContext:
    runAsNonRoot: true  # Security best practice
    runAsUser: 1000
```
Key Options:
- serviceMonitorSelector: {} → Auto-discovers all ServiceMonitors.

- podMonitorSelector: {} → Auto-discovers all PodMonitors.

- serviceAccountName: prometheus → Grants permissions to scrape targets.

## Step 7:Deploy 3 Nginx Services with Metrics
Why?
- We need a workload to monitor (3 Nginx replicas).

- Nginx doesn’t expose metrics by default—we enable them.
```bash
# Deploy Nginx
kubectl create deploy nginx --image=nginx --replicas=3 -n monitoring

# Expose the service
kubectl expose deploy nginx --port=80 -n monitoring

# Enable the built-in Nginx status page (metrics)
kubectl patch deploy nginx -n monitoring --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args", "value": ["-nginx-status-path=/metrics"]}]'
```
What Happens?
- Nginx now exposes metrics at http://<pod-ip>:80/metrics.

## What if WE Want to do All Deploy Ngnix in Step 7 in YAml file?
Deploy Nginx with Metrics via YAML
1. **1. First, create a ConfigMap to enable Nginx metrics**
```bash
# nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: monitoring
data:
  nginx.conf: |
    user  nginx;
    worker_processes  auto;

    error_log  /var/log/nginx/error.log notice;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        # Enable stub status module for metrics
        server {
            listen 80;
            server_name localhost;

            location /metrics {
                stub_status on;
                access_log off;
                allow all;
            }

            location / {
                root   /usr/share/nginx/html;
                index  index.html index.htm;
            }
        }
    }
```
2. **Create the Nginx Deployment (3 replicas)**
```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: monitoring
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "80"
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
```          
3. **Create a Service for Nginx**
```bash
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: monitoring
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
```
## Step 8:Create ServiceMonitor for Nginx    
```bash
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: nginx  # Matches the Nginx Service
  endpoints:
  - port: "80"      # Scrape port
    path: /metrics  # Metrics endpoint
    interval: 15s   # Scrape frequency
```
Key Options:
- selector.matchLabels → Links to the Nginx Service.

- path: /metrics → Where Nginx exposes metrics.

- interval: 15s → How often to scrape.

## Step 9:Configure Node Monitoring
Why?
- Nodes need a DaemonSet (node-exporter) to expose CPU, memory, disk metrics.

- PodMonitor tells Prometheus how to scrape node-exporter.
```bash
# Check if node-exporter is already installed (OLM may deploy it)
kubectl get daemonset -n monitoring -l app.kubernetes.io/name=node-exporter

# If not, deploy manually:
kubectl apply -f node-exporter.yaml
kubectl apply -f node-exporter-podmonitor.yaml
```
node-exporter.yaml

```bash
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter  # Required for selector matching
spec:
  selector:
    matchLabels:
      app: node-exporter  # Must match template.metadata.labels
  template:
    metadata:
      labels:
        app: node-exporter  # Critical: Must match selector.matchLabels
    spec:
      hostNetwork: true  # Access host network namespace
      hostPID: true      # Access host process namespace
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        ports:
        - containerPort: 9100
          name: metrics
        # Security context (recommended)
        securityContext:
          runAsNonRoot: true
          runAsUser: 65534  # nobody user
      # Tolerations to run on master nodes (if needed)
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
```
node-exporter-podmonitor.yaml
```bash
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: node-exporter
  podMetricsEndpoints:
  - port: metrics  # Matches the containerPort
    interval: 15s
```
What Happens?
- node-exporter runs on every node (DaemonSet).

- PodMonitor configures scraping for Prometheus.    
## Verify Monitoring
```bash
# Access Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-operated 9090
```
- Go to http://localhost:9090/targets → Check:

- nginx-monitor (3/3 up)

- node-exporter (1/1 per node)

## Troubleshooting
1. **No Metrics?**
Check Prometheus logs:
```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus-operator
```
- Verify ServiceMonitor/PodMonitor selectors match labels.

2. **Prometheus Not Scraping?**
Check Prometheus config:
```bash
kubectl exec -n monitoring prometheus-main-0 -- cat /etc/prometheus/config_out/prometheus.env.yaml
```
