# CommonLabels in Kustomize

## Overview

`commonLabels` is a field in `kustomization.yaml` that automatically injects labels into **all resources** managed by Kustomize. However, it has been **deprecated** since Kustomize v5 and replaced by the `labels` field.

---

## How CommonLabels Works

When you define `commonLabels`, Kustomize injects the labels into **three places** of every resource:

```
1. metadata.labels
2. spec.selector.matchLabels
3. spec.template.metadata.labels
```

### Example

```yaml
# kustomization.yaml
labels:
  - pairs:
      app: my-app
      env: dev
      # team: ???

```

Kustomize transforms your Deployment into:

```yaml
# Result after kustomize build
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: my-app       # ← injected
    env: dev          # ← injected
spec:
  selector:
    matchLabels:
      app: my-app     # ← injected
      env: dev        # ← injected
  template:
    metadata:
      labels:
        app: my-app   # ← injected
        env: dev      # ← injected
```

---

## The Problem with CommonLabels

Because `commonLabels` injects into `spec.selector.matchLabels`, it causes a critical issue:

> **`spec.selector` is immutable in Kubernetes** — once a Deployment is created, the selector cannot be changed.

### Scenario

```
First deploy  → selector: { app: my-app, env: dev }   ✅ Created
Change label  → selector: { app: my-app, env: prod }  ❌ Immutable error
```

### Error Message

```
Deployment.apps "my-deployment" is invalid:
spec.selector: Invalid value: {"matchLabels":{"app":"my-app","env":"dev"}}: field is immutable
```

---

## Deprecated — Use `labels` Instead

Since Kustomize v5, `commonLabels` is deprecated. The replacement is `labels` with fine-grained control.

### Warning Message

```
Warning: 'commonLabels' is deprecated. Please use 'labels' instead.
Run 'kustomize edit fix' to update your Kustomization automatically.
```

---

## Migration: CommonLabels → Labels

```yaml
# ❌ Old way (deprecated)
commonLabels:
  app: my-app
  env: dev

# ✅ New way
labels:
  - pairs:
      app: my-app
      env: dev
    includeSelectors: false    # do NOT inject into selector
    includeTemplates: true     # inject into template.metadata.labels
```

### `includeSelectors` Options

| Option | Injects into selector | Safe |
|---|---|---|
| `includeSelectors: true` | ✅ Yes | ❌ Risk of immutable error |
| `includeSelectors: false` | ❌ No | ✅ Safe |

---

## Best Practice

### Rule: Never use labels to manage selector

`selector` and `template.labels` should always be defined in `base/deployment.yaml` — never rely on `commonLabels` or `labels` in `kustomization.yaml` to set them.

```yaml
# ✅ base/deployment.yaml — define selector explicitly
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  selector:
    matchLabels:
      app: my-app           # ← defined here, never changes
  template:
    metadata:
      labels:
        app: my-app         # ← must match selector
```

```yaml
# ✅ kustomization.yaml — only add extra metadata labels
labels:
  - pairs:
      env: dev
      managed-by: argocd
    includeSelectors: false   # ← safe, does not touch selector
```

---

## When to Use `labels` in Kustomization

Use it only for **metadata labels** that do not affect selector — such as tagging resources for monitoring, cost tracking, or tooling.

```yaml
labels:
  - pairs:
      team: backend
      cost-center: engineering
      monitored-by: prometheus
    includeSelectors: false
```

---

## Summary

| Feature | `commonLabels` | `labels` |
|---|---|---|
| Status | ❌ Deprecated | ✅ Recommended |
| Injects into selector | ✅ Always | ⚙️ Configurable |
| Risk of immutable error | ✅ High | ❌ Low (if `includeSelectors: false`) |
| Fine-grained control | ❌ No | ✅ Yes |

> **Key Takeaway:** Always define `selector` and `template.labels` in your base Deployment. Use `labels` in `kustomization.yaml` only for additional metadata labels with `includeSelectors: false`.