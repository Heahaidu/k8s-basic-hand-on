# Install

Kubernetes: https://kubernetes.io/docs/tasks/tools/
Minikube: https://minikube.sigs.k8s.io/docs/start/

# Basic command

**CRUD commands**
``` bash
kubectl create deployment [NAME] --image=[IMAGE]
```

``` bash
kubectl edit deployment [NAME]
```

``` bash
kubectl delete deployment [NAME]
```

**Status commands**
``` bash
kubectl get pods|pod|replicaset|deployment|ns|all [-h] [-o [?]]
```

**Debugging**
```bash
kubectl logs [POD_NAME]
```

```bash
kubectl exec -it [POD_NAME] -- bin/bash
```

```bash
kubectl describe pod [POD_NAME]
```

```bash
kubectl describe service [POD_NAME]
```

**Use configuration file to CRUD**

```bash
kubectl apply -f [FILE]
```

```bash
kubectl delete -f [FILE]
```

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
    type: RollingUpdate             # Rolling updates replace Pods gradually (default)
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

# Notes:
# - Clients inside the cluster reach it via: my-app-service:80
# - targetPort points to the containerPort (or named port) on selected Pods
```

## Secret

```YAML
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secret               # Secret name
  namespace: default
type: Opaque                        # Generic key/value secret (common default)/kubernetes.io/tls
stringData:
  API_KEY: "replace-me"             # stringData accepts plain text; API server stores it base64-encoded
  DB_PASSWORD: "replace-me-too"

# Notes:
# - "data:" would require base64-encoded values.
# - Prefer using external secret managers in production when possible.
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
# - Mount ConfigMap as files or consume via environment variables.
```
     
## Ingress

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
spec:
  ingressClassName: nginx          # Specify which Ingress Controller to use
                                   # Common values: nginx, traefik, alb (AWS)

  # --- TLS / HTTPS (Optional) ---
  # tls:
  # - hosts:
  #     - myapp.example.com        # Domain to apply TLS
  #   secretName: myapp-tls-secret # Secret containing tls.crt and tls.key

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

## Presistent Volume

```bash
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
```

## Storage class: [template](./presistent-volume/storage-class.yaml)

# Namespace

## Tools install

kubectx & kubens: https://kubectx.org/

GUI: ```?```

**Switch**
```kubectx [NAMESPACE]```

# Ingress

minikube turn ON ingress: ```minikube addons enable ingress```