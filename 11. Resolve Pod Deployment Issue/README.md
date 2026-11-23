# Kubernetes Troubleshooting Lab: Resolve Pod Deployment Issue (`ErrImagePull`)

## Overview
This lab demonstrates how to diagnose and fix a multi-container Kubernetes Pod failing to start due to an `ErrImagePull` error caused by a typo in the container image tag.

---

## Lab Scenario & Requirements
* **Pod Name:** `webserver` (in the `default` namespace)
* **Main Container:** `nginx-container` (expected image: `nginx:latest`)
* **Sidecar Container:** `sidecar-container` (image: `ubuntu:latest`)
* **Goal:** Identify why the Pod is failing to start, rectify the configuration issue, and ensure the Pod reaches the `Running` (`2/2` Ready) state.

---

## Troubleshooting & Resolution Steps

### 1. Check Pod Status
Inspect the running Pods in the cluster to identify the failure state:
```bash
kubectl get pods
```
**Observation:**
```text
NAME        READY   STATUS         RESTARTS   AGE
webserver   1/2     ErrImagePull   0          45s
```
Only 1 of the 2 containers (`sidecar-container`) is ready, while `nginx-container` is stuck in `ErrImagePull`.

### 2. Inspect the Pod Configuration
Check the Pod specification and container statuses to pinpoint the exact error message:
```bash
kubectl get pod webserver -o yaml | grep -A 5 image
# Or use:
kubectl describe pod webserver
```
**Root Cause:**
The `nginx-container` image was misconfigured with a typo (`nginx:latests` instead of `nginx:latest`), resulting in:
```text
rpc error: code = NotFound desc = failed to pull and unpack image
```

### 3. Fix the Image Tag
Because container `image` fields can be updated in-place on a running Pod without deleting and recreating the Pod, update the image directly using `kubectl set image`:
```bash
kubectl set image pod/webserver nginx-container=nginx:latest
```
*(Alternative: Run `kubectl edit pod webserver`, change `nginx:latests` to `nginx:latest`, and save).*

### 4. Verify the Resolution
Monitor the Pod status until the image pulls and both containers are running:
```bash
kubectl get pods
```
**Expected Output:**
```text
NAME        READY   STATUS    RESTARTS   AGE
webserver   2/2     Running   0          2m
```

---

## Key Takeaways
* **`ErrImagePull` / `ImagePullBackOff`:** Usually indicates a misspelled image name/tag, a non-existent tag in the container registry, or missing image pull credentials (`imagePullSecrets`).
* **Live Pod Updates:** While most Pod `spec` fields are immutable after creation, container `image` tags can be patched live using `kubectl set image` or `kubectl edit`.
