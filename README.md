# Monitoring Kubernetes with Telegraf and InfluxDB

This tutorial will show you how you can deply Telegraf using two methods to monitor your Kubernetes Cluser:

1. Telegraf Helm + Prometheus metrics scraping
2. Telegraf Sidecar for monitoring indurvidual pods + applications

***Assumption: This tutorial will take you through the process of setting up your own local Kubernetes cluster with Minikube. Users with there own clusters can skip these steps and move straight into Telegraf configurations.***

## Prerequisites

 - [ ] (Optional) Install either [Minikube](https://minikube.sigs.k8s.io/docs/start/) or [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/). 
 - [ ] Install [helm](https://helm.sh/docs/intro/install/)
 - [ ] Install [InfluxDB OSS](https://portal.influxdata.com/downloads/) or create a [free cloud account](https://www.influxdata.com/products/influxdb-cloud/)


## Minikube Setup

1. Start a new minikube cluster:

```bash
minikube start  --kubernetes-version=v1.21.1
```

2. Enable metrics-server addon:

```bash
minikube addons enable metrics-server
```

3. Create a namepace called prod (production):

```bash
kubectl create namespace prod  
```


## Telegraf Helm + Promatheus Setup
### Add Helm Repositories

1. Add the InfluxData Helm repo:

```bash
helm repo add influxdata https://helm.influxdata.com/  
```

2. Add the Prometheus-community Helm repo:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts  
```

3. Update Helm repo:

```bash
helm repo update
```

###  Prometheus Metrics Endpoints

1. Install Prometheus Kube State Metrics helm:

```bash
helm upgrade --namespace=kube-system  --install kube-state prometheus-community/kube-state-metrics 
```

2. (Optional) Install Prometheus 

```bash
helm upgrade --namespace=kube-system  --install node-exporter prometheus-community/prometheus-node-exporter 
``` 

### Update Telegraf Configuration 

1. Open [telegraf-kube-min.yml](insert) and add your InfluxDB paramaters:

```yml
outputs:
- influxdb_v2:
	urls:
	 - "<INSERT_INFLUXDB_URL>"
	organization: "<INSERT_ORG>"
	token: "<INSERT_TOKEN>"
	bucket: "Kubernetes"
```

2. Save changes to file

### Deploy Telegraf Helm

1. Deploy Telegraf Helm:

```
helm upgrade --namespace=kube-system  --install telegraf-prometheus -f ./HelmConfig/telegraf-kube-min.yml influxdata/telegraf
```

## Deploy Telegraf Sidecar

### Update Telegraf Operator Classes

1. Open [telegraf-operator-classes.yml](insert) and add your InfluxDB paramaters:

```yml
influxdb_v2: |+
	[[outputs.influxdb_v2]]
		urls = ["<INSERT_INFLUXDB_URL>"]
		token = "<INSERT_TOKEN>"
		organization = "<INSERT_ORG>"
		bucket = "Kubernetes"
		timeout = "5s"
		metric_batch_size = 10000
		metric_buffer_limit = 100000
	[global_tags]
		hostname = "$HOSTNAME"
		nodename = "$NODENAME"
```

2. Save changes to file

### Deploy Sidecar template

1. It's important to deploy the Telegraf Operator Classes first:

```bash
kubectl apply  -f Sidecar/telegraf-operator-classes.yml
```

2.  Deploy the Telegraf Operator

```bash
kubectl apply  -f Sidecar/telegraf-operator.yml
```

## Example: Node-Red + Sidecar Deployment

### Updating the Telegraf Config

1. (Optional) Open [node-red.yml](insert)

2. (Optional) Update the Telegraf configuration with any input plugins you would like to include:

```yml
annotations:
	telegraf.influxdata.com/inputs: |+
		[[inputs.cpu]]
		percpu = true
		totalcpu = true
		collect_cpu_time = false
		report_active = false
		[[inputs.disk]]
		ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs"]
		[[inputs.diskio]]
		[[inputs.kernel]]
		[[inputs.mem]]
		[[inputs.net]]
		[[inputs.processes]]
		[[inputs.swap]]
		[[inputs.system]]
	telegraf.influxdata.com/class: influxdb_v2
	telegraf.influxdata.com/env-fieldref-NAMESPACE: metadata.namespace
	telegraf.influxdata.com/limits-cpu: '750m'
	# invalid memory limit, which will be ignored
	telegraf.influxdata.com/limits-memory: '800x'
```

### Deploy Node-Red

1.  Deploy the Telegraf Operator:

```bash
kubectl apply  -f Prod/node-red.yml
```

2. To access Node-Red you must expose its endpoint. Open a new terminal:

```bash
minikube tunnel 
```

3. Node-Red will be available at http://localhost:1880

## InfluxDB Template

TODO

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

Please make sure to update tests as appropriate.