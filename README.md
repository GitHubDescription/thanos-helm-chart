# thanos helm chart

Helm chart that install and configure prometheus-operator and thanos.

## Goal

Make it as easy and reproducible as possible to setup a high-available and
durable prometheus on kubernetes using prometheus-operator and thanos.

## Getting started

#### 0. Helm repository

```sh
helm repo add carlos https://carlos-charts.storage.googleapis.com
helm repo update
```

#### 1. Namespace

```sh
kubectl create namespace thanos
```

#### 2. Storage config

You'll need a thanos storage config yaml file, as per
[documentation](https://thanos.io/storage.md/).

Then craete a `thanos-storage-config.yaml` file based on the provided
`thanos-storage-config.yaml.example`.

#### 3. Values file

Create a `values.yaml` file based on the provided `values.yaml.example`.
You can check all option available at `./thanos/values.yaml`, as well as
on the official `prometheus-operator` and `grafana` Helm charts.

#### 4. Install

```sh
helm install --namespace thanos --name thanos carlos/thanos -f values.yaml \
  --set-file objectStore=thanos-storage-config.yaml
```

##### 4.1 Upgrade when needed

```sh
helm upgrade --namespace thanos thanos carlos/thanos -f values.yaml \
  --set-file objectStore=thanos-storage-config.yaml
```

#### 5. Port-forward services

You can then port-forward the services you want:

```sh
kubectl -n thanos port-forward svc/thanos-query-http 8080:10902
kubectl -n thanos port-forward svc/thanos-prometheus-operator-prometheus 9090:9090
kubectl -n thanos port-forward svc/thanos-grafana 3000:80
# etc...
```

---

## Multi-cluster architecture

On a multi-cluster (global) architecture the idea is to deploy the "full thing""
on a "central cluster", and only prometheus, alertmanager (?), thanos sidecar
and thanos query on all others:

![](examples/arch.png)

If you want to play with it locally without having 2 "real clusters" to mess
with, you can follow this steps to deploy 2 workloads on 2 different
namespaces: `thanos1` and `thanos2`.

I recommend using `k3s` as it uses less resources and its fast to set up
with `k3d`.

```sh
# create a cluster with k3d
k3d create -n multiple-thanos --wait 0

# fix kubeconfig
export KUBECONFIG="$(k3d get-kubeconfig --name='multiple-thanos')"

# setup helm/tiller
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller --upgrade --wait

# install thanos1
helm upgrade --install --namespace thanos1 thanos1 ./thanos \
	--wait \
	--timeout 600 \
	-f examples/values1.yaml \
	--set-file objectStore=thanos-storage-config.yaml

# install thanos2
helm upgrade --install --namespace thanos2 thanos2 ./thanos \
	--wait \
	--timeout 600 \
	-f examples/values2.yaml \
	--set-file objectStore=thanos-storage-config.yaml
```

Finally, verify you `thanos1` query services can see both `thanos1` store and
sidecar as well as `thanos2` query:

```sh
kubectl -n thanos1 port-forward svc/thanos1-query-http 8080:10902

open http://localhost:8080
```

---

## TODO

- [x] remove peering options (deprecated on thanos 0.4.0+)
- [x] remove some not very useful options, leave sane defaults
- [x] servicemonitors for thanos components
- [ ] TLS
  - [ ] improve config
- [x] service discovery inside the cluster
- [x] service discovery across clusters
- [ ] dynamic prometheus replica label for deduplication
- [x] recommended rules for thanos components
- [x] recommended dashboards for thanos components
- [x] thanos as datasource in grafana
- [x] objectstore config as file
- [ ] ingresses?
- [x] repo/releases
