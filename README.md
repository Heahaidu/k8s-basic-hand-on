# Kubernetes Cheat Sheet

## Mục lục

- [Cài đặt](#cài-đặt)
- [Các câu lệnh](#các-câu-lệnh)
  - [CRUD commands](#crud-commands)
  - [Status commands](#status-commands)
  - [Debugging](#debugging)
  - [Sử dụng file cấu hình để CRUD](#sử-dụng-file-cấu-hình-để-crud)
  - [kubectl config](#kubectl-config)
  - [Các lệnh đặc biệt khác](#các-lệnh-đặc-biệt-khác)
- [YAML Configuration File](#yaml-configuration-file)
  - [Deployment](#deployment)
  - [Service](#service)
  - [Secret](#secret)
  - [ConfigMap](#configmap)
  - [Persistent Volume / PVC / StorageClass](#persistent-volume--pvc--storageclass)
  - [Ingress](#ingress)
  - [Namespace](#namespace)
  - [ResourceQuota / LimitRange](#resourcequota--limitrange)
- [Namespace](#namespace-management)
- [Tài nguyên khác (bonus)](#tài-nguyên-khác-bonus)

---

# Cài đặt

Kubernetes: https://kubernetes.io/docs/tasks/tools/
Minikube: https://minikube.sigs.k8s.io/docs/start/

---

# Các câu lệnh

## CRUD commands

```bash
kubectl create deployment [NAME] --image=[IMAGE]
```

```bash
kubectl edit deployment [NAME]
```

```bash
kubectl delete deployment [NAME]
```

```bash
kubectl scale deployment [NAME] --replicas=[N]      # tăng/giảm số replica
```

```bash
kubectl rollout restart deployment [NAME]           # restart toàn bộ pod (không downtime nếu rolling)
```

```bash
kubectl rollout undo deployment [NAME]              # rollback về version trước
```

## Status commands

```bash
kubectl get pods|pod|replicaset|deployment|ns|all [-h] [-o wide|json|yaml]
```

```bash
kubectl get pods -w                                  # watch, theo dõi thay đổi real-time
```

```bash
kubectl get pods --sort-by=.metadata.creationTimestamp
```

```bash
kubectl get events --sort-by=.lastTimestamp          # xem event gần nhất (rất hữu ích khi debug)
```

```bash
kubectl top pod|node                                 # CPU/Memory usage, cần cài metrics-server
```

```bash
kubectl rollout status deployment [NAME]             # theo dõi tiến trình rollout
```

```bash
kubectl rollout history deployment [NAME]            # xem lịch sử các revision
```

## Debugging

```bash
kubectl logs [POD_NAME]
```

```bash
kubectl logs -f [POD_NAME]                           # follow log, giống tail -f
```

```bash
kubectl logs [POD_NAME] -c [CONTAINER_NAME]          # pod có nhiều container
```

```bash
kubectl exec -it [POD_NAME] -- /bin/bash
```

```bash
kubectl exec -it [POD_NAME] -c [CONTAINER_NAME] -- /bin/sh   # khi pod có nhiều container
```

```bash
kubectl describe pod [POD_NAME]
```

```bash
kubectl describe service [SERVICE_NAME]
```

```bash
kubectl describe node [NODE_NAME]
```

```bash
kubectl get pod [POD_NAME] -o yaml                   # xem full spec đang chạy thực tế trên cluster
```

## Sử dụng file cấu hình để CRUD

```bash
kubectl apply -f [FILE]
```

```bash
kubectl delete -f [FILE]
```

```bash
kubectl diff -f [FILE]                               # xem trước những gì sẽ thay đổi trước khi apply
```

```bash
kubectl apply -k [DIRECTORY]                         # apply bằng Kustomize
```

## kubectl config

Quản lý cluster/context/namespace đang làm việc (lưu trong file `~/.kube/config`).

```bash
kubectl config view                                  # xem toàn bộ kubeconfig
```

```bash
kubectl config get-contexts                          # liệt kê các context có sẵn
```

```bash
kubectl config current-context                       # context đang sử dụng
```

```bash
kubectl config use-context [CONTEXT_NAME]            # chuyển sang cluster/context khác
```

```bash
kubectl config set-context --current --namespace=[NAMESPACE]   # đổi namespace mặc định cho context hiện tại
```

```bash
kubectl config delete-context [CONTEXT_NAME]
```

```bash
kubectl config rename-context [OLD_NAME] [NEW_NAME]
```

## Các lệnh đặc biệt khác

```bash
kubectl port-forward [POD_NAME] 8080:80              # forward port từ máy local vào pod
```

```bash
kubectl cp [POD_NAME]:/path/in/pod ./local/path      # copy file giữa local và pod
```

```bash
kubectl label pod [POD_NAME] key=value               # gắn label cho resource
```

```bash
kubectl annotate pod [POD_NAME] key=value            # gắn annotation cho resource
```

```bash
kubectl explain [RESOURCE].[FIELD]                   # xem doc field trong spec, ví dụ: kubectl explain pod.spec.containers
```

```bash
kubectl autoscale deployment [NAME] --min=2 --max=5 --cpu-percent=80   # tạo HPA nhanh
```

```bash
kubectl cordon [NODE_NAME]                           # đánh dấu node không cho schedule pod mới
```

```bash
kubectl drain [NODE_NAME]                            # di dời pod ra khỏi node, dùng trước khi maintenance
```

```bash
kubectl uncordon [NODE_NAME]                         # cho phép schedule lại trên node
```

```bash
kubectl taint nodes [NODE_NAME] key=value:NoSchedule # đánh taint để chặn pod không có toleration phù hợp
```

---

# YAML Configuration File

## Deployment

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app                      # Deployment name (must be unique within the namespace)
  namespace: default                # Optional; if omitted, uses the "default" namespace
  labels:
    app: my-app                     # Labels help group and select resources
spec:
  replicas: 2                       # Desired number of Pod replicas
  revisionHistoryLimit: 10          # How many old ReplicaSets to keep for rollback
  strategy:
    type: RollingUpdate             # RollingUpdate (default) hoặc Recreate (xóa hết pod cũ rồi mới tạo pod mới)
    rollingUpdate:
      maxSurge: 1                   # Extra Pods allowed above desired replicas during update
      maxUnavailable: 0             # Pods allowed to be unavailable during update
  selector:
    matchLabels:
      app: my-app                   # MUST match template.metadata.labels (immutable after creation)
  template:
    metadata:
      labels:
        app: my-app                 # Pod labels (Service selectors usually match these)
    spec:
      # initContainers chạy xong trước khi container chính khởi động (ví dụ: chờ DB sẵn sàng, migrate data)
      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command: ["sh", "-c", "until nc -z my-db 5432; do sleep 1; done"]
      containers:
        - name: my-app
          image: nginx:1.27         # Container image (use a pinned tag for reproducibility)
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080   # Port the container listens on (informational + used by probes)
          env:
            - name: APP_MODE
              value: "prod"         # Plain env var
            - name: CONFIG_VALUE
              valueFrom:
                configMapKeyRef:
                  name: my-app-config
                  key: FEATURE_FLAG # Read a single key from a ConfigMap
            - name: SECRET_VALUE
              valueFrom:
                secretKeyRef:
                  name: my-app-secret
                  key: API_KEY      # Read a single key from a Secret
          resources:
            requests:
              cpu: 100m             # Minimum CPU guaranteed for scheduling
              memory: 128Mi         # Minimum memory guaranteed for scheduling
            limits:
              cpu: 500m             # CPU hard limit (exceeding leads to throttling)
              memory: 256Mi         # Memory hard limit (exceeding may OOMKill the container)
          livenessProbe:             # Restarts container if it becomes unhealthy
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:            # Controls whether Pod receives traffic from Services
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
      # --- Điều khiển pod chạy trên node nào (optional) ---
      # nodeSelector:
      #   disktype: ssd             # Pod chỉ được schedule vào node có label disktype=ssd
      # tolerations:
      #   - key: "key"
      #     operator: "Equal"
      #     value: "value"
      #     effect: "NoSchedule"    # Cho phép pod chạy trên node đã bị taint tương ứng
      # affinity:
      #   podAntiAffinity:          # Ví dụ: không cho 2 pod cùng app chạy chung 1 node
      #     requiredDuringSchedulingIgnoredDuringExecution:
      #       - labelSelector:
      #           matchLabels:
      #             app: my-app
      #         topologyKey: "kubernetes.io/hostname"
```

## Service

```YAML
apiVersion: v1
kind: Service
metadata:
  name: my-app-service              # Service name (stable network identity)
  namespace: default
  labels:
    app: my-app
spec:
  type: ClusterIP                   # ClusterIP (internal), NodePort, LoadBalancer, ExternalName
  selector:
    app: my-app                     # Routes traffic to Pods with this label
  ports:
    - name: http
      protocol: TCP                 # TCP or UDP
      port: 80                      # Service port (clients connect to this)
      targetPort: http              # Pod port (can be a number or a named container port)
      # nodePort: 30080              # Chỉ dùng khi type: NodePort (range 30000-32767)

# Notes:
# - Clients inside the cluster reach it via: my-app-service:80
# - targetPort points to the containerPort (or named port) on selected Pods
# - Headless Service (không cần Cluster IP, trả thẳng IP của từng Pod):
#   spec.clusterIP: None -> dùng khi cần DNS resolve trực tiếp tới từng pod (StatefulSet)

# So sánh các loại Service:
# ┌──────────────┬──────────────────────────────┬───────────────────────────┐
# │ Type         │ Use case                      │ Access URL                │
# ├──────────────┼──────────────────────────────┼───────────────────────────┤
# │ ClusterIP    │ Internal only (default)        │ service-name:port         │
# │ NodePort     │ Dev/testing, expose qua node   │ NodeIP:30000-32767        │
# │ LoadBalancer │ Expose qua external IP (cloud) │ ExternalIP:port           │
# │ ExternalName │ Map service tới DNS bên ngoài  │ CNAME tới external host   │
# └──────────────┴──────────────────────────────┴───────────────────────────┘
```

## Secret

```YAML
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secret               # Secret name
  namespace: default
type: Opaque                        # Generic key/value secret (common default)
stringData:
  API_KEY: "replace-me"             # stringData accepts plain text; API server stores it base64-encoded
  DB_PASSWORD: "replace-me-too"

# Notes:
# - "data:" would require base64-encoded values (echo -n 'value' | base64).
# - Prefer using external secret managers in production when possible (Vault, AWS Secrets Manager, Sealed Secrets,...).

# Các type Secret hay dùng:
# ┌────────────────────────────────────┬──────────────────────────────────────────┐
# │ type                                │ Use case                                  │
# ├────────────────────────────────────┼──────────────────────────────────────────┤
# │ Opaque                              │ Key/value tự do (mặc định)                │
# │ kubernetes.io/tls                   │ Cert + key cho TLS (tls.crt, tls.key)     │
# │ kubernetes.io/dockerconfigjson      │ Credentials để pull image từ private repo │
# │ kubernetes.io/basic-auth            │ username/password                         │
# └────────────────────────────────────┴──────────────────────────────────────────┘

# Tạo nhanh bằng imperative command (không cần file YAML):
# kubectl create secret generic my-app-secret --from-literal=API_KEY=replace-me
# kubectl create secret docker-registry my-regcred --docker-username=USER --docker-password=PASS --docker-server=SERVER

# Inject toàn bộ Secret vào container dưới dạng env (thay vì từng key):
#   envFrom:
#     - secretRef:
#         name: my-app-secret
```

## ConfigMap

```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config               # ConfigMap name
  namespace: default
data:
  FEATURE_FLAG: "true"              # Simple key/value config
  APP_CONFIG_YAML: |                # Multiline config (stored as a single value)
    server:
      port: 8080
    logging:
      level: info

# Notes:
# - Use ConfigMap for non-sensitive config.
# - Tạo nhanh từ file: kubectl create configmap my-app-config --from-file=app.conf

# Cách 1 - Inject toàn bộ ConfigMap vào container dưới dạng env:
#   envFrom:
#     - configMapRef:
#         name: my-app-config

# Cách 2 - Mount ConfigMap thành file trong container (hữu ích cho config dạng file):
#   volumes:
#     - name: config-volume
#       configMap:
#         name: my-app-config
#   volumeMounts:
#     - name: config-volume
#       mountPath: /etc/config       # mỗi key trong data sẽ thành 1 file trong thư mục này
```

## Persistent Volume / PVC / StorageClass

```YAML
# ======================
# pv.yaml
# ======================
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: local
spec:
  # Total storage capacity of this volume
  capacity:
    storage: 1Gi

  # Access modes:
  #   - ReadWriteOnce (RWO): single node can read/write
  #   - ReadOnlyMany  (ROX): multiple nodes can read only
  #   - ReadWriteMany (RWX): multiple nodes can read/write
  accessModes:
    - ReadWriteOnce

  # What happens when PVC is deleted:
  #   - Retain:  keep data, PV must be manually cleaned
  #   - Delete:  remove PV and its data automatically
  #   - Recycle: wipe data but keep PV (deprecated)
  persistentVolumeReclaimPolicy: Retain

  # StorageClass name — PVC must use the same name to bind
  storageClassName: manual

  # Storage backend — using hostPath (local directory on node)
  # Other options: nfs, awsElasticBlockStore, gcePersistentDisk, etc.
  hostPath:
    path: "/mnt/data"

---
# ======================
# pvc.yaml
# ======================
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  # Must match one of the PV's accessModes
  accessModes:
    - ReadWriteOnce

  # Amount of storage requested
  # Must be <= PV's capacity to bind successfully
  resources:
    requests:
      storage: 500Mi

  # Must match PV's storageClassName to bind
  storageClassName: manual

---
# ======================
# pod.yaml
# ======================
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: app
      image: nginx
      # Mount the volume into the container's filesystem
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"  # path inside container
          name: my-storage                     # must match volumes[].name below

  # Define volumes for this pod
  volumes:
    - name: my-storage
      # Reference the PVC by name
      persistentVolumeClaim:
        claimName: my-pvc  # must match PVC metadata.name

---
# ======================
# storage-class.yaml
# ======================
# StorageClass dùng để CẤP PHÁT PV TỰ ĐỘNG (dynamic provisioning) thay vì tạo PV tay như trên.
# Khi PVC tạo ra mà chưa có PV nào khớp, provisioner của StorageClass sẽ tự tạo PV mới.
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs   # tùy cloud: kubernetes.io/gce-pd, kubernetes.io/azure-disk, rancher.io/local-path,...
parameters:
  type: gp3                          # tham số riêng theo provisioner (ví dụ loại disk)
reclaimPolicy: Delete                # Delete (default) hoặc Retain
allowVolumeExpansion: true           # cho phép resize PVC sau khi tạo
volumeBindingMode: WaitForFirstConsumer  # chỉ tạo volume khi có Pod thực sự cần dùng (tránh tạo ở sai zone)

# Lưu ý:
# - Nếu dùng StorageClass dynamic, PVC chỉ cần khai báo storageClassName: fast-ssd, KHÔNG cần tạo PV tay.
# - Minikube có sẵn StorageClass "standard" (provisioner: k8s.io/minikube-hostpath).
```

---

## Namespace

```YAML
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace                # Tên namespace (DNS-1123 label: chữ thường, số, dấu -)
  labels:
    env: production                 # Label để filter, ví dụ: kubectl get ns -l env=production
    team: backend
  annotations:
    description: "Namespace cho hệ thống backend"

# Tạo nhanh bằng lệnh (không cần file):
# kubectl create namespace my-namespace

# Notes:
# - Namespace dùng để cô lập resource theo môi trường (dev/staging/prod) hoặc theo team/project.
# - Một số resource KHÔNG thuộc namespace (cluster-scoped): Node, PersistentVolume, StorageClass, ClusterRole,...
# - Namespace mặc định có sẵn: default, kube-system, kube-public, kube-node-lease.
# - Xóa namespace sẽ xóa luôn TẤT CẢ resource bên trong (Deployment, Service, Secret,...).
```

## ResourceQuota / LimitRange 

```YAML
# ======================
# resourcequota.yaml — giới hạn tổng resource được dùng trong namespace
# ======================
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-namespace-quota
  namespace: my-namespace
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"                      # số pod tối đa trong namespace

---
# ======================
# limitrange.yaml — default/min/max resource cho từng container nếu không khai báo resources
# ======================
apiVersion: v1
kind: LimitRange
metadata:
  name: my-namespace-limits
  namespace: my-namespace
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "2"
        memory: 1Gi
      min:
        cpu: 50m
        memory: 64Mi
```

# Namespace Management

## Tools install

kubectx & kubens: https://kubectx.org/

GUI:
- [Lens](https://k8slens.dev/) — IDE quản lý cluster trực quan, xem log/exec/resource ngay trên UI.
- [K9s](https://k9scli.io/) — terminal UI (tương tác bằng bàn phím, nhẹ, không cần GUI).
- [Kubernetes Dashboard](https://github.com/kubernetes/dashboard) — dashboard web chính thức của Kubernetes.

**Switch**
```bash
kubectx [NAMESPACE]
```

**Các lệnh cơ bản với namespace**
```bash
kubectl create namespace [NAME]
```

```bash
kubectl get namespace
```

```bash
kubectl delete namespace [NAME]      # sẽ xóa luôn TẤT CẢ resource trong namespace đó
```

```bash
kubectl get all -n [NAMESPACE]       # xem resource trong 1 namespace cụ thể
```

> Tip: dùng `kubectl config set-context --current --namespace=[NAME]` (phần [kubectl config](#kubectl-config) ở trên) để không phải gõ `-n [NAMESPACE]` mỗi lần.

# Ingress

```YAML
# Ingress: Manages external HTTP/HTTPS access to Services in the cluster
# Acts as a reverse proxy / API gateway (like Nginx, Traefik, HAProxy)
# Requires an Ingress Controller to be installed (e.g., nginx-ingress, traefik)
# On Minikube: minikube addons enable ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress             # Name of the Ingress resource
  annotations:
    # Annotations depend on which Ingress Controller you use
    # Below are common annotations for Nginx Ingress Controller
    nginx.ingress.kubernetes.io/rewrite-target: /          # Rewrite URL path to /
    # nginx.ingress.kubernetes.io/ssl-redirect: "false"    # Disable HTTPS redirect
    # nginx.ingress.kubernetes.io/proxy-body-size: "10m"   # Max upload size
    # cert-manager.io/cluster-issuer: "letsencrypt-prod"   # Tự động cấp TLS cert qua cert-manager
spec:
  ingressClassName: nginx          # Specify which Ingress Controller to use
                                   # Common values: nginx, traefik, alb (AWS)

  # --- TLS / HTTPS (Optional) ---
  tls:
    - hosts:
        - myapp.example.com        # Domain to apply TLS
      secretName: myapp-tls-secret # Secret type kubernetes.io/tls chứa tls.crt và tls.key

  # --- Routing Rules ---
  rules:
  # Rule 1: Route traffic based on hostname
  - host: myapp.example.com       # Domain name (must point to cluster IP via DNS or /etc/hosts)
                                   # If omitted, rule applies to ALL incoming traffic
    http:
      paths:
      # Path 1: Route /app to my-app-service
      - path: /app
        pathType: Prefix           # Prefix: matches /app, /app/*, /app/anything
                                   # Exact:  matches only /app (not /app/something)
        backend:
          service:
            name: my-app-service   # Name of the target Service
            port:
              number: 80           # Port of the target Service (the 'port' field in Service)

      # Path 2: Route /api to another service
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000

  # Rule 2: Different hostname -> different service
  - host: dashboard.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dashboard-service
            port:
              number: 8080

  # --- Default Backend (Optional) ---
  # Handles requests that don't match any rule above (like a 404 fallback)
  # defaultBackend:
  #   service:
  #     name: fallback-service
  #     port:
  #       number: 80

# Traffic flow:
#
# Browser: myapp.example.com/app
#   -> Ingress Controller (Nginx Pod listening on port 80/443)
#     -> Ingress rules (match host + path)
#       -> Service: my-app-service:80
#         -> Pod: targetPort
#
# Comparison with other access methods:
# ┌──────────────┬────────────────────────────────┬──────────────────────────┐
# │ Method       │ Use case                       │ Access URL               │
# ├──────────────┼────────────────────────────────┼──────────────────────────┤
# │ ClusterIP    │ Internal only                  │ service-name:port        │
# │ NodePort     │ Dev/testing                    │ NodeIP:30000-32767       │
# │ LoadBalancer │ One service per external IP    │ ExternalIP:port          │
# │ Ingress      │ Multiple services, one IP      │ domain.com/path          │
# │              │ Host/path-based routing        │                          │
# │              │ TLS termination, rewrites      │                          │
# └──────────────┴────────────────────────────────┴──────────────────────────┘
```

```bash
minikube addons enable ingress       # bật Ingress Controller (nginx) trên Minikube
```

---

# Tài nguyên khác (bonus)

Những resource này không có trong yêu cầu ban đầu nhưng thường gặp khi làm việc thực tế với K8s — nói ngắn để bạn biết tên mà tra/đụng vào không bị lạ, mình có thể viết thêm full YAML nếu cần:

- **Job** — chạy 1 task tới khi xong thì dừng (ví dụ: migrate DB, batch job).
- **CronJob** — như Job nhưng chạy theo schedule (cron expression), ví dụ backup hàng ngày.
- **StatefulSet** — như Deployment nhưng dành cho app cần định danh ổn định + storage riêng cho mỗi pod (database, Kafka, Elasticsearch).
- **DaemonSet** — đảm bảo mỗi node trong cluster đều có 1 pod chạy (ví dụ: log collector, monitoring agent).
- **HorizontalPodAutoscaler (HPA)** — tự scale số replica của Deployment theo CPU/Memory (xem `kubectl autoscale` ở trên).
- **NetworkPolicy** — quy định pod nào được phép giao tiếp với pod nào (firewall ở cấp pod).
- **ServiceAccount / Role / RoleBinding (RBAC)** — quản lý quyền truy cập API server cho pod/user.