# HyperFlow Storage deployment on Kubernetes

## Using Google Kubernetes Engine

Here are steps you need to do in order to run Ceph on the Google Kubernetes Engine.

### Configure `gcloud` and create Kubernetes cluster

- Install the `gcloud` client as described [here](https://cloud.google.com/sdk/install).
- If needed, [create a new project](https://cloud.google.com/resource-manager/docs/creating-managing-projects) using the Google cloud console.
```bash
gcloud projects create "$PROJECT_NAME"
```
- Create the Kubernetes cluster with the following commands (set the `$PROJECT_ID`, `$CLUSTER_NAME`, and `$ZONE` variables):

```bash
gcloud container clusters create "$CLUSTER_NAME" \
    --project "$PROJECT_ID" \
    --zone "$ZONE" \
    --release-channel "rapid" \
    --machine-type "n1-standard-2" \
    --image-type "UBUNTU_CONTAINERD" \
    --num-nodes "3" \
    --node-labels "nodetype=worker" \
    --enable-stackdriver-kubernetes \
    --enable-ip-alias \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver

gcloud container clusters get-credentials "$CLUSTER_NAME" --zone "$ZONE"

gcloud container node-pools create "hfmaster-pool" \
    --project "$PROJECT_ID" \
    --zone "$ZONE" \
    --cluster "$CLUSTER_NAME" \
    --machine-type "n1-standard-2" \
    --image-type "UBUNTU_CONTAINERD" \
    --num-nodes "1" \
    --node-labels "nodetype=hfmaster"

gcloud container node-pools create "storage-pool" \
    --project "$PROJECT_ID" \
    --zone "$ZONE" \
    --cluster "$CLUSTER_NAME" \
    --machine-type "n1-standard-1" \
    --image-type "UBUNTU_CONTAINERD" \
    --num-nodes "3" \
    --node-labels "nodetype=storage"
```

- [Install `kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and configure `kubeconfig` to access your GKE cluster [following these instructions](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#generate_kubeconfig_entry).

- [Install `helm`](https://helm.sh/docs/intro/install/)


### Monitoring Deployment

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```bash
kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    -f storage/values/kube-prometheus-stack.yaml
```

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 8000:80
# http://127.0.0.1:8000
# user: admin
# password: prom-operator
```

### Rook Deploymnet

```bash
# https://github.com/rook/rook/blob/master/Documentation/helm-operator.md
helm repo add rook-release https://charts.rook.io/release
```

```bash
kubectl create namespace rook-ceph

helm dependency update storage/charts/hyperflow-storage

helm install hyperflow-storage storage/charts/hyperflow-storage \
    --namespace rook-ceph \
    -f storage/values/hyperflow-storage.yaml
```

```
kubectl port-forward service/rook-ceph-mgr-dashboard 8443 -n rook-ceph
# https://127.0.0.1:8443
# user: admin
# Get password
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
    -o jsonpath="{['data']['password']}" | base64 --decode && echo
```
