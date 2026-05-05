## Install Prometheus-Operator and Grafana

```
kube-prometheus-stack
  ├── Prometheus
  ├── Grafana (+ dashboard có sẵn)
  ├── Alertmanager
  ├── kube-state-metrics
  ├── node-exporter
  ├── Prometheus Operator
  └── Pre-built dashboards + alerting rules
```

### Add repo

``` bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Install chart

``` bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace <namespace>
```

### Install chart with fixed version

``` bash
helm install prometheus prometheus-community/kube-prometheus-stack --version "9.4.1" -- namespace <namespace>
```

---

## Install Proometheus

```
prometheus
  ├── Prometheus server
  ├── Alertmanager
  ├── kube-state-metrics
  ├── node-exporter
  └── pushgateway
```

``` bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Install chart

``` bash
helm install prometheus prometheus-community/prometheus --namespace <namespace>
```

### Install chart with fixed version

``` bash
helm install prometheus prometheus-community/prometheus --version "9.4.1" -- namespace <namespace>
```