Đây là hành vi đúng theo thiết kế của Kubernetes scheduler + Auto Mode/Karpenter, dù nhìn có vẻ "lãng phí". Vài điểm giải thích:

## 1. Pod phân bố không multi-AZ đều — do `topologySpreadConstraints` hoặc không có constraint

Mặc định, nếu Deployment/Helm chart **không khai báo** `topologySpreadConstraints` hoặc `podAntiAffinity`, scheduler chỉ quan tâm **resource fit**, không cố ý spread đều theo AZ. Kết quả: 1 node 1 pod, node khác 3 pod — hoàn toàn tùy vào lúc pod được tạo và node nào có resource available tại thời điểm đó.

`kube-prometheus-stack` có sẵn 1 số constraint cho riêng Prometheus/Alertmanager (anti-affinity giữa các replica), nhưng các pod khác (operator, grafana, kube-state-metrics...) thường không có.

## 2. Vì sao tạo node mới dù node cũ còn dư resource

Đây là do **scheduling timing**, không phải Autoscaler "quyết định sai":

```
Thời điểm T0: Pod A, B, C được tạo → chưa có node → Karpenter tạo Node 1 (size phù hợp 3 pod)
Thời điểm T1: Pod D được tạo → Node 1 ĐANG full/đang pending từ T0 
              → Karpenter thấy pod D pending → tạo Node 2 ngay
Thời điểm T2: Node 1 xong, có resource dư → nhưng pod D ĐÃ được schedule vào Node 2 rồi
```

→ Karpenter/Autoscaler **react theo pending pod tại thời điểm đó**, không "nhìn trước" rằng Node 1 sẽ sớm có dư resource. Sau khi node 2 đã chạy, Kubernetes **không tự di chuyển pod** giữa node để tối ưu lại (không có cơ chế "rebalance" tự động trừ khi dùng thêm tool).

## 3. Vì sao 1 node c6g.large vẫn đủ cho 4 pod, nhưng đôi khi tạo 2 node

`c6g.large` = 2 vCPU, 4GB RAM. Nếu pod request nhỏ (ví dụ mỗi pod request 0.25 vCPU/256Mi), 4 pod tổng = 1 vCPU/1GB — node dư nhiều. Nhưng nếu các pod được **tạo gần như đồng thời** (như lúc `helm install` apply hết manifest cùng lúc), scheduler có thể:
- Schedule 3 pod đầu vào Node 1 (đang provision).
- Pod thứ 4 đến lúc Node 1 chưa `Ready` → bị pending → Karpenter spawn Node 2 ngay.

Đây là **race condition giữa tốc độ tạo pod và tốc độ Node 1 join cluster** (Node mới cần ~1-2 phút để Ready).

## Cách giảm tình trạng này (nếu muốn tối ưu cost)

1. **Consolidation** (Karpenter feature, Auto Mode hỗ trợ): tự động terminate node thừa và reschedule pod sang node khác khi có thể. Kiểm tra NodePool config có enable consolidation chưa.

2. **Pod Disruption Budget + giảm số replica** nếu không cần HA cao cho dev/test (ví dụ Prometheus operator chỉ cần 1 replica).

3. **Chấp nhận** — nếu cluster nhỏ/dev, 1-2 node dư resource không đáng kể về cost (vài $ một ngày), không cần tối ưu quá sâu.

→ Tóm lại: đây là **hành vi bình thường** của hệ thống scheduling + autoscaling, không phải bug. Việc pod phân bố "lệch" và node dư resource là tradeoff giữa **tốc độ phản hồi pending pod** và **tối ưu chi phí tuyệt đối** — Karpenter ưu tiên cái đầu.

Có, nhưng cần phân biệt rõ: **node group không tự giải quyết vấn đề này** — vấn đề là do **scheduler + autoscaler**, không phải do cách provision node (Auto Mode vs node group). Tuy nhiên, dùng `aws_eks_node_group` + công cụ khác sẽ cho bạn **nhiều "núm điều chỉnh" hơn**.

## 1. Cluster Autoscaler vs Karpenter — cấu hình chi tiết hơn

Với node group, bạn dùng **Cluster Autoscaler**, có config rõ ràng hơn:

```yaml
# values cho cluster-autoscaler helm chart
extraArgs:
  scale-down-enabled: true
  scale-down-delay-after-add: 5m
  scale-down-unneeded-time: 5m
  scale-down-utilization-threshold: 0.5  # node dưới 50% utilization sẽ bị xem xét scale down
  balance-similar-node-groups: true       # cân bằng pod giữa các node group tương tự
  skip-nodes-with-system-pods: false
```

→ `scale-down-utilization-threshold` + `scale-down-unneeded-time` giúp **chủ động dồn pod lại** và xóa node thừa sau một khoảng thời gian — đây là cái Auto Mode/Karpenter cũng có nhưng ít expose config hơn.

## 2. `topologySpreadConstraints` — kiểm soát phân bố pod theo AZ

Đây là cấu hình ở **level Deployment/pod**, áp dụng được cho cả node group VÀ Auto Mode:

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule  # hoặc ScheduleAnyway
      labelSelector:
        matchLabels:
          app: my-app
```

→ Đảm bảo pod luôn spread đều giữa AZ — bạn cần **thêm cái này vào Helm chart values** (nhiều chart như `kube-prometheus-stack` hỗ trợ `topologySpreadConstraints` qua `values.yaml`).

## 3. `Descheduler` — rebalance pod sau khi đã chạy

Add-on riêng (không phải addon AWS, là open-source K8s descheduler), chạy định kỳ để **di chuyển pod** từ node ít resource sang node nhiều resource hơn — giải quyết đúng vấn đề "node 1 dư resource nhưng pod đã nằm ở node 2".

```bash
helm install descheduler descheduler/descheduler -n kube-system
```

Strategy hữu ích: `LowNodeUtilization` — tự balance lại pod giữa các node.

## 4. Bin-packing — đặt resource request chính xác

Nếu pod request quá cao so với thực tế dùng → scheduler tính sai capacity, dễ tạo thêm node không cần thiết. Dùng VPA (Vertical Pod Autoscaler) ở mode "recommendation" để biết request hợp lý.

---

## Tóm lại

| Giải pháp | Node group cần? | Auto Mode hỗ trợ? |
|---|---|---|
| Cluster Autoscaler config chi tiết | Có | Không (Auto Mode dùng cơ chế riêng, ít config) |
| `topologySpreadConstraints` | Không cần — pod-level | Có |
| Descheduler | Không cần | Có |
| Consolidation (Karpenter-style) | Cần tự cài Karpenter | Built-in nhưng ít tùy chỉnh |

→ **Node group + Cluster Autoscaler** cho bạn nhiều control hơn về threshold/timing scale-down. Nhưng để fix vấn đề **pod phân bố lệch AZ**, `topologySpreadConstraints` ở pod-level mới là giải pháp đúng — áp dụng được dù bạn dùng node group hay Auto Mode.