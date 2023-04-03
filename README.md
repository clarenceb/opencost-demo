Open Cost demo
==============

Create an AKS cluster
---------------------

```sh
# Set these to your specific values
CLUSTER_NAME=opencost
CLUSTER_RESOURCE_GROUP=opencost
CLUSTER_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
LOCATION=australiasoutheast

az group create -n $CLUSTER_RESOURCE_GROUP -l $LOCATION

AKS_LATEST_VERSION=$(az aks get-versions -l $LOCATION --query orchestrators[*].orchestratorVersion -o tsv | sort -nr | head -n1)
echo "Kubernetes version: $AKS_LATEST_VERSION"

az aks create \
    -n $CLUSTER_NAME \
    -g $CLUSTER_RESOURCE_GROUP \
    -l $LOCATION \
    --node-count 2 \
    --network-plugin azure \
    -k $AKS_LATEST_VERSION \
    --enable-managed-identity \
    -a monitoring \
    --generate-ssh-keys

az aks get-credentials --resource-group $CLUSTER_RESOURCE_GROUP --name $CLUSTER_NAME --overwrite-existing
kubectl config use-context $CLUSTER_NAME
```

Installation
------------

Follow the official [installation guide](https://www.opencost.io/docs/install).

```sh
# Install Prometheus
helm install my-prometheus --repo https://prometheus-community.github.io/helm-charts prometheus \
  --namespace prometheus --create-namespace \
  --set pushgateway.enabled=false \
  --set alertmanager.enabled=false \
  --set serviceMonitor.enable=true \
  -f https://raw.githubusercontent.com/opencost/opencost/develop/kubernetes/prometheus/extraScrapeConfigs.yaml

# Install OpenCost
kubectl apply --namespace opencost -f https://raw.githubusercontent.com/opencost/opencost/develop/kubernetes/opencost.yaml
```

Workload deploy
---------------

Deploy some workloads into a couple of namespaces to allocate resources:

```sh
kubectl apply -k overlays/app-a
kubectl apply -k overlays/app-b
```

View OpenCost web interface
---------------------------

```sh
kubectl port-forward --namespace opencost service/opencost 9003 9090
```

You may access the OpenCost UI at [http://localhost:9090](http://localhost:9090)

Vierify server is running: [http://localhost:9003/allocation/compute?window=60m](http://localhost:9003/allocation/compute?window=60m)

Install kubectl-cost plugin
---------------------------

See: https://www.opencost.io/docs/kubectl-cost

```sh
kubectl krew install cost
```

Use the kubectl-cost plugin
---------------------------

```sh
kubectl port-forward --namespace opencost service/opencost 9003 9090

alias kcac='kubectl cost --service-port 9003 --service-name opencost --kubecost-namespace opencost --allocation-path /allocation/compute'

# Show the projected monthly rate for each namespace with all cost components displayed.
kcac namespace --show-all-resources

# kubectl cost --service-port 9003 --service-name opencost --kubecost-namespace opencost --allocation-path /allocation/compute  \
kcac namespace \
    --historical \
    --window 7d \
    --show-cpu \
    --show-memory \
    --show-pv \
    --show-efficiency=false

# Find the total projected monthly costs based on the last two hours:
kcac namespace \
    --window 2h \
    --show-efficiency=true

# Show the projected monthly rate for each controller based on the last 7 days of activity with PV (persistent volume) cost breakdown.
kcac controller --window 7d --show-pv

# Show costs over the past 7 days broken down by the value of the app label:
kcac label  --window 7d --historical -l app

# Show the projected monthly rate for each deployment based on the last month of activity with CPU, memory, GPU, PV, and network cost breakdown.
kcac deployment --window month -A

# Show the projected monthly rate for each deployment in the kubecost namespace based on the last 3 days of activity with CPU cost breakdown.
kcac deployment \
  --window 3d \
  --show-cpu \
  -n kubecost

# Show how much each pod in the "kube-system" namespace cost yesterday, including CPU-specific cost.
kcac pod \
  --historical \
  --window yesterday \
  --show-cpu \
  -n kube-system

# Alternatively, kubectl cost can show cost by the asset type. To view node cost with breakdowns of RAM and CPU cost for a window of 7 days.
kcac node \
  --historical \
  --window 7d \
  --show-cpu \
  --show-memory
```

Run OpenCost as a Prometheus Metrics Exporter
---------------------------------------------

```sh
kubectl apply --namespace opencost-exporter \
-f https://raw.githubusercontent.com/opencost/opencost/develop/kubernetes/exporter/opencost-exporter.yaml
```

Prometheus interface
--------------------

```sh
kubectl port-forward -n prometheus svc/my-prometheus-server 8080:80

# Access http://localhost:8080 in your browser
```

Try some [example queries](https://www.opencost.io/docs/prometheus#example-queries).

Total CPU Cost per month:

```promql
sum(
    avg(kube_node_status_capacity_cpu_cores) by (node) * avg(node_cpu_hourly_cost) by (node) * 730 +
    avg(node_gpu_hourly_cost) by (node) * 730
)
```

CPU Cost per node per month:

```promql
avg(kube_node_status_capacity_cpu_cores) by (node) * avg(node_cpu_hourly_cost or up * 0) by (node) * 730
```

Grafana interface
-----------------

```sh
export POD_NAME=$(kubectl get pods --namespace default -l "ap
p.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].
metadata.name}")

# Get Grafana admin user password
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --d
ecode ; echo

kubectl --namespace default port-forward $POD_NAME 3000

# Access http://localhost:3000 in your browser and login as u: admin, p: <see-above>
```

Use this dashboard: https://grafana.com/grafana/dashboards/10217-kubecost-cluster-metrics/

```sh
kubectl edit cm  my-prometheus-server -n Prometheus
```

Change `opencost` job to `kubecost` job.

Edit any fields (e.g. totals) to use numeric fields.

Resources
---------

* https://www.opencost.io/docs/
