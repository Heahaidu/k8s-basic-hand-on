# cAdvisor Missing `container` và `image` Labels trên Kubernetes v1.35

## Triệu chứng

Khi scrape metrics từ `/metrics/cadvisor` qua Prometheus, tất cả các metrics có dạng:

```
container_cpu_usage_seconds_total{container="", image="", namespace="default", pod="nginx-xxx"} 0.5
container_memory_working_set_bytes{container="", image="", ...} 104857600
```

Label `container` và `image` đều rỗng (`""`), khiến các Prometheus rule, Grafana dashboard, và alert không hoạt động đúng.

---

## Nguyên nhân

### Kiến trúc của vấn đề

Vấn đề xảy ra khi Kubernetes dùng **cri-dockerd** làm container runtime (thường gặp ở Minikube với driver mặc định).

```
┌─────────────────────────────────────────────────────────┐
│  kubelet  ──→  cri-dockerd.sock  ──→  Docker daemon     │
│                (CRI interface)        (tạo containers)   │
│                                                          │
│  cAdvisor ──→  docker.sock       ──→  Docker daemon      │
│                (Docker API)           "No such container"│
└─────────────────────────────────────────────────────────┘
```

**Chi tiết:**

1. kubelet tạo container thông qua **cri-dockerd** (CRI shim), container ID được sinh ra bởi cri-dockerd.
2. cAdvisor (built-in trong kubelet) tự kết nối vào **docker.sock** để query metadata.
3. Docker daemon không nhận ra container ID do cri-dockerd sinh ra → trả về `"No such container"`.
4. cAdvisor không lấy được tên container và image → label bị để trống.

### Log lỗi điển hình

```
kubelet[2170]: "DeleteContainer returned error" err="failed to get container status: 
rpc error: code = Unknown desc = Error response from daemon: No such container: abc123..."

cadvisor_stats_provider.go:569] "Partial failure issuing cadvisor.ContainerInfoV2" 
err="partial failures: [\"/kubepods/besteffort/pod...\": RecentStats: unable to find data in memory cache]"
```

### Tại sao containerd/cri-o không bị lỗi này?

cAdvisor có các **factory** riêng để detect runtime theo thứ tự ưu tiên:

| Runtime    | Socket path                        | cAdvisor support |
|------------|------------------------------------|-----------------|
| containerd | `/run/containerd/containerd.sock`  | ✅ Native        |
| cri-o      | `/run/crio/crio.sock`              | ✅ Native        |
| docker     | `/var/run/docker.sock`             | ⚠️ Docker API   |
| cri-dockerd| `/var/run/cri-dockerd.sock`        | ❌ Không detect  |

Với **containerd** và **cri-o**, kubelet và cAdvisor dùng **chung một socket** → container ID khớp hoàn toàn → metadata đầy đủ.

```
┌─────────────────────────────────────────────────────────┐
│  kubelet  ──→  containerd.sock  ──→  containerd runtime  │
│                                                          │
│  cAdvisor ──→  containerd.sock  ──→  containerd runtime  │
│                (cùng socket!)         metadata đầy đủ ✅ │
└─────────────────────────────────────────────────────────┘
```

### Bối cảnh lịch sử

- **Kubernetes 1.20**: Docker runtime bị deprecated.
- **Kubernetes 1.24**: Dockershim bị remove khỏi kubelet. cri-dockerd ra đời như một shim bên ngoài.
- **Kubernetes 1.35**: cri-dockerd vẫn hoạt động nhưng cAdvisor không hỗ trợ đúng cách.

---

## Cách xác nhận lỗi

```bash
# Kiểm tra container runtime đang dùng
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'

# Kiểm tra metrics có labels không
kubectl get --raw /api/v1/nodes/<node-name>/proxy/metrics/cadvisor | \
  grep 'container_cpu_usage_seconds_total' | \
  grep -v 'container=""' | \
  grep -v 'container="POD"' | \
  head -5

# Nếu không có output → labels đang bị thiếu
```

```bash
# Kiểm tra config kubelet
cat /var/lib/kubelet/config.yaml | grep containerRuntimeEndpoint
# containerRuntimeEndpoint: unix:///var/run/cri-dockerd.sock  ← đây là nguyên nhân
```

---

## Cách fix

### Fix dứt điểm: Đổi container runtime sang containerd

Đây là cách duy nhất thực sự giải quyết vấn đề.

#### Với Minikube

```bash
# Xóa cluster cũ
minikube delete

# Tạo lại với containerd
minikube start \
  --kubernetes-version=v1.35.0 \
  --container-runtime=containerd \
  --driver=docker        # hoặc --driver=hyperv tùy môi trường
```

#### Với kubeadm

Trong file cấu hình kubelet `/var/lib/kubelet/config.yaml`:

```yaml
containerRuntimeEndpoint: "unix:///run/containerd/containerd.sock"
```

Đảm bảo containerd đã được cài đặt và đang chạy:

```bash
systemctl status containerd
systemctl enable --now containerd
```

#### Verify sau khi fix

```bash
kubectl get --raw /api/v1/nodes/minikube/proxy/metrics/cadvisor | \
  grep 'container_cpu_usage_seconds_total' | \
  grep -v 'container=""' | \
  grep -v 'container="POD"' | \
  head -5
```

Output mong đợi:

```
container_cpu_usage_seconds_total{container="coredns",image="registry.k8s.io/coredns/coredns:v1.12.0",namespace="kube-system",pod="coredns-xxx",...} 1.234
```

---

### Workaround tạm thời (không khuyến nghị)

Nếu không thể đổi runtime, dùng Prometheus relabeling để join với `kube-state-metrics`:

```promql
# Join để lấy lại image info từ kube-state-metrics
container_cpu_usage_seconds_total
* on(namespace, pod, container)
group_left(image)
kube_pod_container_info
```

Hoặc lọc bỏ metrics thiếu label trong scrape config:

```yaml
# prometheus.yml
metric_relabel_configs:
  - source_labels: [container]
    regex: ".+"   # Chỉ giữ metrics có container label không rỗng
    action: keep
```

> ⚠️ Workaround này chỉ ẩn vấn đề, không fix được gốc rễ. Một số metrics node-level và pod-level aggregate sẽ bị mất.

---

## Cài đặt kube-prometheus-stack

Sau khi fix runtime, cài Prometheus stack:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Tạo namespace
kubectl create namespace prometheus

# Install (CRDs được install tự động)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace prometheus \
  --create-namespace \
  --timeout 10m \
  --wait
```

Nếu gặp lỗi CRDs chưa tồn tại:

```bash
# Install CRDs thủ công trước
helm pull prometheus-community/kube-prometheus-stack --untar
kubectl apply --server-side -f kube-prometheus-stack/charts/crds/crds/

# Sau đó install chart
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace prometheus \
  --skip-crds
```

---

## Tóm tắt

| | cri-dockerd | containerd | cri-o |
|---|---|---|---|
| cAdvisor detect | ❌ | ✅ | ✅ |
| `container` label | ❌ rỗng | ✅ đầy đủ | ✅ đầy đủ |
| `image` label | ❌ rỗng | ✅ đầy đủ | ✅ đầy đủ |
| Kubernetes 1.35 support | ⚠️ | ✅ | ✅ |

**Kết luận:** Dùng `--container-runtime=containerd` hoặc `--container-runtime=cri-o` khi khởi tạo cluster để đảm bảo cAdvisor hoạt động đúng với Prometheus.