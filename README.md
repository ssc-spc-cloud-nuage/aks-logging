# Rancher logging
Setup localhost Rancher with logging using Fluentd, Elasticsearch and Kibana.

# Setup
1. Install [docker](https://docs.docker.com/get-docker/), [helm](https://helm.sh/), [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [k3d](https://github.com/rancher/k3d).
1. Create a `~/mnt` directory.

# Rancher
```sh
# Create the local cluster
k3d cluster create rancher --volume ~/mnt:/mnt --port 443:443@loadbalancer --port 80:80@loadbalancer

# Add the helm repos
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest && \
helm repo add jetstack https://charts.jetstack.io && \
helm repo update

# Create cert-manager CRDs
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.crds.yaml

# Install cert-manager
kubectl create namespace cert-manager && \
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.15.0 --wait

# Install rancher
kubectl create namespace cattle-system && \
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.k3d.localhost --wait
```

# Elasticsearch and Kibana
```sh
# Add the helm repo
helm repo add elastic https://helm.elastic.co && \
helm repo update

# Create PersistentVolumes 
kubectl apply -f hostpath-pv.yaml --namespace elastic

# Install Elasticsearch
kubectl create namespace elastic && \
helm install elastic elastic/elasticsearch \
	--values ./values-elasticsearch.yaml \
	--namespace elasic \
	--wait
	  
# Forward port for elasticsearch
# kubectl port-forward svc/elasticsearch-master 9200 --namespace elastic

# Install Kibana (front-end)
helm install kibana elastic/kibana \
	--values ./values-kibana.yaml \
	--namespace elasic \
	--wait

# Forward port for Kibana
# kubectl port-forward deployment/kibana-kibana 5601 --namespace elastic

```

# Install Rancher tools
Cluster Explorer > Apps & Marketplace:
1. Monitoring
1. Logging
