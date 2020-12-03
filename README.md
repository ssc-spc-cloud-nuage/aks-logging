# Rancher logging
Setup localhost Rancher with logging using Fluentd, Elasticsearch and Kibana.

# Setup
1. Install [docker](https://docs.docker.com/get-docker/), [helm](https://helm.sh/), [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [k3d](https://github.com/rancher/k3d).
1. Create a `~/mnt/elastic/01` and `~/mnt/elastic/02` directories.

# Rancher
```sh
# Create the local cluster
k3d cluster create rancher --volume ~/mnt:/mnt --port 443:443@loadbalancer --port 80:80@loadbalancer

# Add the helm repos
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest && \
helm repo add jetstack https://charts.jetstack.io && \
helm repo update

kubectl create namespace cert-manager && \
kubectl create namespace cattle-system

# Create cert-manager CRDs
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.crds.yaml

# Install cert-manager
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.1.0 --wait

# Install rancher
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.k3d.localhost --wait
```

# Elasticsearch and Kibana
```sh
# Add the helm repo
helm repo add elastic https://helm.elastic.co && \
helm repo update

# Create namespace
kubectl create namespace elastic

# Create PersistentVolumes 
kubectl apply -f hostpath-pv.yaml --namespace elastic

# Install Elasticsearch
helm install elastic elastic/elasticsearch \
	--values ./values-elasticsearch.yaml \
	--namespace elastic \
	--wait

# Install Kibana (front-end)
helm install kibana elastic/kibana \
	--values ./values-kibana.yaml \
	--namespace elastic \
	--wait

# Create the rancher-logging index in Elasticsearch
curl -X PUT http://elasticsearch.k3d.localhost/rancher-logging

```

# Install Rancher tools
Cluster Explorer > Apps & Marketplace:
1. Logging

# Create ClusterFlow and ClusterOutput
Elasticsearch config for an output flow:
```sh
URL: http://elasticsearch-master.elastic.svc.cluster.local:9200
Index: rancher-logging
```

# Access
* https://rancher.k3d.localhost/
* https://elasticsearch.k3d.localhost/
* https://kibana.k3d.localhost/

# Troubleshooting
1. If Elasticsearch is not receiving logs, [troubleshoot Fluentd](https://banzaicloud.com/docs/one-eye/logging-operator/operation/troubleshooting/fluentd/).
1. View the [Elasticsearch indicies](http://elasticsearch.k3d.localhost/_cat/indices?bytes=b&s=store.size:desc&v) to check if `rancher-logging` exists and is growing in size.
1. Increase the Fluentd log level, restart it and view logs:
```sh
kubectl edit loggings.logging.banzaicloud.io rancher-logging
# fluentd:
#    logLevel: debug

kubectl delete pod rancher-logging-fluentd-0
kubectl exec -it rancher-logging-fluentd-0 -- cat /fluentd/log/out
```