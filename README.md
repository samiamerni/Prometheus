# Monitor your cluster security

This repository uses the following applications:
- [Prometheus Stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Grafana](https://grafana.com/)
- [Promtail & Loki](https://grafana.com/oss/loki/)


Here is how to use the resources:

Create a monitoring namespace:
```
kubectl create ns monitoring
```

Install the helm prometheus stack chart:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```
helm upgrade --install prometheus  prometheus-community/kube-prometheus-stack -n monitoring --values prom-values.yaml
```

Expose Pormetheus, Grafana and Alert manager:
```
kubectl expose -n monitoring svc prometheus-kube-prometheus-prometheus  --name prometheus-ext --port 9090 --type NodePort
```
```
kubectl expose -n monitoring svc prometheus-grafana  --name prometheus-grafana-ext --port 3000 --type NodePort
```
```
kubectl -n monitoring describe svc prometheus-kube-prometheus-alertmanager-ext
```

In my case i had some configurations to add into kubernetes components:

## kube-proxy:
I added this line into cm/kube-proxy: metricsBindAddress: 0.0.0.0:10249
```
 kubectl edit cm/kube-proxy -n kube-system
```

## etcd:
I added this ",http://192.168.8.75:2381" into manifest /etc/kubernetes/manifests/etcd.yaml: --listen-metrics-urls=http://127.0.0.1:2381,http://192.168.8.75:2381
```
vim /etc/kubernetes/manifests/etcd.yaml
```
## scheduler:

I changed "--bind-address=127.0.0.1"  by"--bind-address=192.168.8.75" 
And i had to restart kubelet and containerd because kube-scheduler stucks in pending state



Install promtail to collect logs from every node:
```
helm upgrade --install promtail grafana/promtail -f promtail-values.yaml -n monitoring
```


Install loki to collect all the logs from promtail:
```
helm upgrade --install loki grafana/loki-distributed -n monitoring
```

Configure Alert Manager: ( make sure to modify the file )

```
helm upgrade --reuse-values -f alertmanager-config.yaml prometheus prometheus-community/kube-prometheus-stack -n monitoring
```

Configure Alert rules:

```
helm upgrade --reuse-values -f alert-rules.yaml prometheus prometheus-community/kube-prometheus-stack -n monitoring
```