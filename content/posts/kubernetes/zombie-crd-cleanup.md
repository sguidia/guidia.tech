---
title: "How to Delete Zombie CRDs in Kubernetes"
date: 2025-07-05
menu:
  sidebar:
    name: How to Delete Zombie CRDs in Kubernetes
    identifier: zombie-crd-cleanup
    parent: kubernetes
    weight: 10
summary: "Sometimes Kubernetes CRD objects refuse to die—even when their namespace is long gone. Here's how to exorcise these zombie resources."
---

> **Scenario:** The Namespace is already deleted, yet a CRD object (e.g. `MyResource`) still shows up in the cluster and cannot be removed with normal `kubectl` commands.

---

## 1. The Problem

- `kubectl get <crd> --all-namespaces` lists an object although its Namespace no longer exists.
- `kubectl get` / `kubectl delete` returns **`NotFound`** or hangs in `Terminating`.
- Controller logs might say: `cannot delete <kind> … still referenced`.

Such an object is called a **“zombie”**: it’s marked for deletion (`deletionTimestamp`), but never finalized due to missing Namespace or controller.

---

## 2. Why “Zombies” Happen

| Cause | What Actually Happens |
|-------|------------------------|
| **Namespace deleted before controller cleared finalizer** | The `metadata.finalizers` list remains. No controller → no cleanup. |
| **Helm or Argo CD with `prune: true`** | The CRD object is removed from the manifest, then the Namespace disappears. |
| **API-server/network outage** | An interruption mid-deletion leaves the object dangling in etcd. |

---

## 3. Detecting Orphaned CRD Objects

```bash
kubectl get myresources.example.com --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}  {"→"}  {.metadata.deletionTimestamp}{"\\n"}{end}'
```

### Red Flags

- `deletionTimestamp` is set.
- `metadata.finalizers` is **not empty**.
- `kubectl get ns <namespace>` → **NotFound**.

---

## 4. How to Clean Up

| Method                       | Use When                                | Example |
|-----------------------------|------------------------------------------|---------|
| Normal delete                | Namespace exists, controller is healthy | `kubectl delete myresource foo -n live-ns` |
| Patch away the finalizer    | Deletion stuck, controller present       | `kubectl patch myresource foo -n live-ns -p '{"metadata":{"finalizers":[]}}' --type=merge` |
| Force delete                | Controller missing                       | `kubectl delete myresource foo -n live-ns --grace-period=0 --force` |
| Recreate Namespace temporarily | Namespace is gone                     | See below |
| Raw API `/finalize`         | Extreme edge case                        | See below |

---

### 4.1 Reliable Workflow (Namespace Re-creation)

```bash
# 1. Recreate the ghost namespace
kubectl create namespace ghost-ns

# 2. Remove the finalizer
kubectl patch myresource orphan -n ghost-ns \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

# 3. Force delete the CR
kubectl delete myresource orphan -n ghost-ns --grace-period=0 --force

# 4. Remove the temporary namespace
kubectl delete namespace ghost-ns
```

---

### 4.2 Using Raw API to `/finalize` (No Namespace Required)

```bash
kubectl proxy &

curl -k -H 'Content-Type: application/json' -X PUT \
  --data '{"apiVersion":"example.com/v1","kind":"MyResource","metadata":{"finalizers":[]}}' \
  http://127.0.0.1:8001/apis/example.com/v1/namespaces/ghost-ns/myresources/orphan/finalize
```

---

## 5. Verify

```bash
kubectl get myresources.example.com --all-namespaces
# Zombie object should be gone.
```

---

## 6. Takeaways

- Zombies happen when finalizers can’t run—often due to a missing controller or Namespace.
- Either patch the finalizer or hit the `/finalize` endpoint.
- Best practice: watch for `deletionTimestamp` + non-empty `finalizers`.

You now have a reliable strategy to eliminate ghost CRD objects from your Kubernetes clusters.
