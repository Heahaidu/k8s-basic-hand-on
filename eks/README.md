# EKS Configuration Reference Guide

A practical reference covering node isolation, multi-AZ pod scheduling, ALB setup, and subpath routing for Amazon EKS.

## Table of Contents

1. [Taints and Tolerations — Dedicated Node Groups](#1-taints-and-tolerations--dedicated-node-groups)
2. [Taint Effects Explained](#2-taint-effects-explained)
3. [TopologySpreadConstraints — Multi-AZ for Pods](#3-topologyspreadconstraints--multi-az-for-pods)
4. [Setting Up ALB for EKS](#4-setting-up-alb-for-eks)
5. [Subpath Routing for Services (with Full Annotations)](#5-subpath-routing-for-services-with-full-annotations)

---

## 1. Taints and Tolerations — Dedicated Node Groups

### Purpose

Taints and tolerations let you reserve a node group for a specific workload or namespace, preventing other pods from being scheduled there unless they explicitly tolerate the taint.

```
Taint (on Node)  +  Toleration (on Pod)  +  NodeSelector (on Pod)
       │                    │                       │
  "Reject all"      "I can tolerate this"    "I must run here"
```

### Step 1: Create the node group with a taint and label

```yaml
managedNodeGroups:
  - name: game-2048
    instanceType: t3.small
    desiredCapacity: 2
    minSize: 2
    maxSize: 4
    volumeSize: 20
    privateNetworking: true
    availabilityZones: ["us-east-1a", "us-east-1b"]
    labels:
      workload: game-2048
    taints:
      - key: workload
        value: game-2048
        effect: NoSchedule
```

```bash
eksctl create nodegroup -f cluster.yaml --include=game-2048
```

### Step 2: Apply taint/label manually (if the node group already exists)

```bash
kubectl taint nodes -l eks.amazonaws.com/nodegroup=game-2048 \
  workload=game-2048:NoSchedule

kubectl label nodes -l eks.amazonaws.com/nodegroup=game-2048 \
  workload=game-2048
```

### Step 3: Add toleration and nodeSelector to the Pod spec

```yaml
spec:
  template:
    spec:
      nodeSelector:
        workload: game-2048
      tolerations:
        - key: workload
          operator: Equal
          value: game-2048
          effect: NoSchedule
```

### Why system DaemonSets still appear on the node

`aws-node`, `kube-proxy`, and `eks-pod-identity-agent` are DaemonSets created with `operator: Exists`, which tolerates **every** taint. This is intentional — these provide pod networking, service routing, and IAM credentials, so they must run on every node regardless of custom taints.

```bash
kubectl get daemonset -n kube-system aws-node -o yaml | grep -A 5 tolerations
```

### Verification

```bash
# A pod WITHOUT toleration should stay Pending on tainted nodes
kubectl run test-block --image=nginx -n default
kubectl describe pod test-block | grep -A5 Events

# A pod WITH toleration should land on the correct node
kubectl get pods -n game-2048 -o wide
```

---

## 2. Taint Effects Explained

### Purpose

A taint is not complete without an `effect`.

The effect tells Kubernetes what action to take when a pod does not have a matching toleration.

```yaml
taints:
  - key: workload
    value: game-2048
    effect: NoSchedule
```

In this example:

```text
key      = workload
value    = game-2048
effect   = NoSchedule
```

The scheduler evaluates the effect when deciding whether a pod can run on the node.

---

### NoSchedule

The most common effect.

```yaml
taints:
  - key: workload
    value: game-2048
    effect: NoSchedule
```

Meaning:

> Do not schedule new pods onto this node unless they have a matching toleration.

Example:

```text
Node:
  workload=game-2048:NoSchedule

Pod:
  no toleration
```

Result:

```text
❌ Pod remains Pending
❌ Scheduler rejects the node
```

If the pod contains:

```yaml
tolerations:
  - key: workload
    operator: Equal
    value: game-2048
    effect: NoSchedule
```

Result:

```text
✅ Pod may be scheduled onto the node
```

Use cases:

- Dedicated node groups
- Spot instances
- GPU nodes
- Isolated workloads

---

### PreferNoSchedule

A soft version of NoSchedule.

```yaml
taints:
  - key: workload
    value: game-2048
    effect: PreferNoSchedule
```

Meaning:

> Try to avoid scheduling pods here, but allow it if no better option exists.

Example:

```text
Node A -> PreferNoSchedule
Node B -> Normal
```

Scheduler behavior:

```text
Prefer Node B
```

If Node B becomes unavailable:

```text
Node B full
```

Then:

```text
✅ Scheduler may place the pod on Node A
```

Think of it as:

```text
NoSchedule       = Hard restriction
PreferNoSchedule = Soft preference
```

Use cases:

- Gradual workload migration
- Temporary workload segregation
- Non-critical node preferences

---

### NoExecute

The strongest effect.

```yaml
taints:
  - key: workload
    value: game-2048
    effect: NoExecute
```

Meaning:

1. Do not schedule new pods.
2. Evict existing pods that do not tolerate the taint.

Example:

```text
Node:
  workload=game-2048:NoExecute

Running Pods:
  pod-a
  pod-b
```

If neither pod has a matching toleration:

```text
❌ pod-a evicted
❌ pod-b evicted
```

If a pod contains:

```yaml
tolerations:
  - key: workload
    operator: Equal
    value: game-2048
    effect: NoExecute
```

Result:

```text
✅ Pod remains on the node
```

Common use cases:

- Node maintenance
- Node draining
- Handling unhealthy nodes

---

### NoExecute with tolerationSeconds

A pod can temporarily tolerate a NoExecute taint.

```yaml
tolerations:
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300
```

Meaning:

```text
Node becomes NotReady
        ↓
Pod stays for 300 seconds
        ↓
Node still unhealthy
        ↓
Pod is evicted
```

This is commonly used by Kubernetes system workloads during temporary node failures.

---

### Summary


| Effect           | New Pods          | Existing Pods              |
| ---------------- | ----------------- | -------------------------- |
| NoSchedule       | Blocked           | Stay Running               |
| PreferNoSchedule | Avoid If Possible | Stay Running               |
| NoExecute        | Blocked           | Evicted (unless tolerated) |


---

### Recommended Production Pattern

For dedicated EKS node groups:

```yaml
labels:
  workload: game-2048

taints:
  - key: workload
    value: game-2048
    effect: NoSchedule
```

Combined with:

```yaml
nodeSelector:
  workload: game-2048

tolerations:
  - key: workload
    operator: Equal
    value: game-2048
    effect: NoSchedule
```

This provides:

```text
✓ Pods must run on the intended node group
✓ Other workloads cannot consume those nodes
✓ Predictable scheduling behavior
```

---

## 3. TopologySpreadConstraints — Multi-AZ for Pods

### Purpose

Having nodes in multiple AZs does **not** guarantee pods spread across them. `topologySpreadConstraints` is required to force the scheduler to distribute pods evenly by zone.

### Prerequisite check

```bash
kubectl get nodes -l workload=game-2048 \
  -o custom-columns=NAME:.metadata.name,ZONE:.metadata.labels.topology\\.kubernetes\\.io/zone
```

Confirm at least 2 nodes exist across 2 different zones before relying on this constraint.

### Full Deployment example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: game-2048
  name: deployment-2048
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  replicas: 2
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      nodeSelector:
        workload: game-2048
      tolerations:
        - key: workload
          operator: Equal
          value: game-2048
          effect: NoSchedule
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: app-2048
      containers:
        - image: public.ecr.aws/l6m2t8p7/docker-2048:latest
          name: app-2048
          ports:
            - containerPort: 80
```

### Common mistakes


| Mistake                                                                   | Correct value                                                       |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `topologyKey: topology.kubernetes.io.zone` (dot)                          | `topology.kubernetes.io/zone` (slash)                               |
| `labelSelector.matchLabels: {app: game-2048}` not matching real pod label | Must exactly match `template.metadata.labels`                       |
| `whenUnsatisfiable: DoNotSchedule` with insufficient nodes per zone       | Use `ScheduleAnyway` while testing, or ensure enough nodes per zone |


### Force re-scheduling of existing pods

Kubernetes does **not** re-evaluate already-running pods against a newly applied constraint. Delete them to trigger a fresh scheduling decision:

```bash
kubectl delete pod -n game-2048 -l app.kubernetes.io/name=app-2048
```

### Verification

```bash
kubectl get pods -n game-2048 -o json | \
  jq -r '.items[] | "\(.metadata.name) -> Node: \(.spec.nodeName)"'

kubectl get nodes -l workload=game-2048 \
  -o custom-columns=NAME:.metadata.name,ZONE:.metadata.labels.topology\\.kubernetes\\.io/zone
```

---

## 4. Setting Up ALB for EKS

### Architecture overview

```
Internet → Route53 (DNS) → ACM (HTTPS) → ALB → Ingress → Service (ClusterIP) → Pod
```

### Step 1: Create IAM Policy for the controller

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.9.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### Step 2: Create IAM Service Account (IRSA)

```bash
eksctl create iamserviceaccount \
  --cluster=test \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-1 \
  --approve
```

### Step 3: Install the controller via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=test \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<VPC_ID>
```

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Step 4: Tag subnets correctly

```bash
# Public subnet — ALB lives here
aws ec2 create-tags --resources <PUBLIC_SUBNET_ID> \
  --tags Key=kubernetes.io/role/elb,Value=1

# Private subnet — nodes/pods live here
aws ec2 create-tags --resources <PRIVATE_SUBNET_ID> \
  --tags Key=kubernetes.io/role/internal-elb,Value=1
```

### Step 5: Request an ACM certificate (for HTTPS)

```bash
aws acm request-certificate \
  --domain-name game.yourdomain.com \
  --validation-method DNS \
  --region us-east-1
```

### Step 6: Service — use ClusterIP with target-type: ip

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
  selector:
    app.kubernetes.io/name: app-2048
```

### Step 7: Ingress with HTTPS and health checks

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:<ACCOUNT_ID>:certificate/<CERT_ID>
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
spec:
  ingressClassName: alb
  rules:
    - host: game.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-2048
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-2048.yaml
kubectl get ingress ingress-2048 -n game-2048
```

### Step 8: Point Route53 to the ALB

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id <HOSTED_ZONE_ID> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "game.yourdomain.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "<ALB_HOSTED_ZONE_ID>",
          "DNSName": "<ALB_DNS_NAME>",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Production hardening checklist


| Item                      | How                                                                                                              |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Readiness/Liveness probes | Add `httpGet` probes matching the real app route                                                                 |
| Resource requests/limits  | Set `resources.requests` / `resources.limits`                                                                    |
| Horizontal Pod Autoscaler | `kubectl autoscale deployment ... --min=2 --max=10 --cpu-percent=70`                                             |
| PodDisruptionBudget       | `minAvailable: 1` to survive node drains                                                                         |
| WAF                       | Attach via `alb.ingress.kubernetes.io/wafv2-acl-arn`                                                             |
| Access logs               | `alb.ingress.kubernetes.io/load-balancer-attributes: access_logs.s3.enabled=true,access_logs.s3.bucket=<bucket>` |


### Verifying IAM Service Account

```bash
# List all IRSA bindings
eksctl get iamserviceaccount --cluster test --region us-east-1

# Inspect the role ARN attached to a ServiceAccount
kubectl get sa aws-load-balancer-controller -n kube-system \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'

# Check attached policies
aws iam list-attached-role-policies --role-name <ROLE_NAME>

# Confirm the pod actually receives AWS credentials
kubectl exec -it <pod-name> -n kube-system -- env | grep AWS
```

---

## 5. Subpath Routing for Services (with Full Annotations)

### Purpose

Route multiple services through a **single shared ALB**, each serving a different path (e.g. `/`, `/api`, `/health`) or host, instead of provisioning a separate load balancer per service.

```
Without group.name:
  Ingress A → ALB-1
  Ingress B → ALB-2     ❌ Extra cost, no sharing

With group.name:
  Ingress A (group: shared-alb) ─┐
                                   ├──→ Same ALB, multiple listener rules
  Ingress B (group: shared-alb) ─┘
```

### Key annotations explained


| Annotation                                   | Purpose                                                                                                                                            |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `alb.ingress.kubernetes.io/group.name`       | Must be **identical** across all Ingress objects sharing the ALB (case-sensitive, no extra whitespace)                                             |
| `alb.ingress.kubernetes.io/group.order`      | Determines rule priority. **Lower number = evaluated first.** Specific paths (e.g. `/api`) must have a lower order than catch-all paths (e.g. `/`) |
| `alb.ingress.kubernetes.io/scheme`           | Must match across all Ingress in the same group (`internet-facing` or `internal`)                                                                  |
| `alb.ingress.kubernetes.io/target-type`      | `ip` routes directly to pod IPs (recommended); should be consistent across the group                                                               |
| `alb.ingress.kubernetes.io/healthcheck-path` | Path the ALB uses to check target health — must match the app's real health endpoint                                                               |


### Example: catch-all path `/` (frontend)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: shared-alb
    alb.ingress.kubernetes.io/group.order: '20'
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-2048
                port:
                  number: 80
```

### Example: specific path `/api` (backend, same ALB)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: api-namespace
  name: ingress-api
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: shared-alb
    alb.ingress.kubernetes.io/group.order: '5'
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '5'
    alb.ingress.kubernetes.io/success-codes: '200'
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /actuator
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

### Important caveats

- **ALB does not strip path prefixes.** Unlike Nginx Ingress, there is no `rewrite-target` annotation. The full path (e.g. `/api/users`) is forwarded as-is to the backend. The backend application must either:
  - handle the prefix itself in its routing code, or
  - sit behind an Nginx sidecar that rewrites the path before proxying internally.
- **Frontend apps with absolute asset paths** (e.g. apps that load `/style.css` rather than `/game/style.css`) will break under a subpath. For these, prefer **host-based routing** (`game.yourdomain.com`) instead of subpath routing, while still sharing the same ALB via `group.name`.
- **Rule ordering matters.** A broad path like `/` will swallow all traffic if its `group.order` is lower than a more specific path like `/api`. Always give catch-all rules a higher order number.

### Verifying that two Ingress objects truly share one ALB

The most reliable check is comparing the `ADDRESS` column — if it matches, it is the same physical ALB:

```bash
kubectl get ingress -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,ADDRESS:.status.loadBalancer.ingress[0].hostname
```

```
NS            NAME              ADDRESS
game-2048     ingress-2048      k8s-sharedalb-xxxx.us-east-1.elb.amazonaws.com
api-namespace ingress-api       k8s-sharedalb-xxxx.us-east-1.elb.amazonaws.com   ← same = same ALB ✅
```

If addresses differ, compare annotations character-by-character:

```bash
kubectl get ingress ingress-2048 -n game-2048 -o jsonpath='{.metadata.annotations}' | jq
kubectl get ingress ingress-api -n api-namespace -o jsonpath='{.metadata.annotations}' | jq
```

### Note on changing group.name on an existing Ingress

The AWS Load Balancer Controller does **not** rename an already-provisioned ALB just because `group.name` was added or changed later. If an Ingress was created before `group.name` existed, you may need to delete and re-apply it so the controller builds a fresh model from scratch:

```bash
kubectl delete ingress ingress-2048 -n game-2048

# confirm the old ALB is gone before re-creating
aws elbv2 describe-load-balancers --region us-east-1 \
  --query "LoadBalancers[?contains(LoadBalancerName,'game2048')]"

kubectl apply -f ingress-2048.yaml
```

### Troubleshooting checklist

```
☐ group.name identical across all Ingress (case-sensitive)
☐ group.order set explicitly — specific paths lower than catch-all paths
☐ scheme and ip-address-type consistent across the group
☐ Service has a populated Endpoint (kubectl get endpoints <svc>)
☐ healthcheck-path matches the app's actual health route
☐ Security Group allows ALB → Node/Pod traffic on the target port
```

```bash
# Check controller logs for reconciliation errors
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=100 | grep -iE "error|warn"

# Check real target health on AWS
aws elbv2 describe-target-health --target-group-arn <TG_ARN> --region us-east-1
```

