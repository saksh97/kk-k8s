# Kubernetes Lab: Time-Check Pod & ConfigMap

This repository contains the solution steps, YAML manifest, and study notes for the **KodeKloud Nautilus DevOps Kubernetes Lab** focusing on **Namespaces**, **ConfigMaps**, **Volumes**, and **Pods**.

---

## 📋 Lab Requirements

1. **Namespace:** Use the `xfusion` namespace.
2. **Pod & Container:** Create a Pod named `time-check` with a container named `time-check` running the `busybox:latest` image.
3. **ConfigMap:** Create a ConfigMap named `time-config` in the `xfusion` namespace with the data `TIME_FREQ=5`.
4. **Command & Environment Variable:**
   - Execute the command: `while true; do date; sleep $TIME_FREQ;done`
   - Write the output to `/opt/sysops/time/time-check.log`
   - Inject `TIME_FREQ` as an environment variable sourced from the `time-config` ConfigMap.
5. **Volume:** Create an `emptyDir` volume named `log-volume` and mount it at `/opt/sysops/time` inside the container.

---

## 🚀 Steps to Complete the Lab

### Step 1: Create the Single Manifest File
Run the following command on the `jump-host` to generate `time-check-all.yaml`, which defines the Namespace, ConfigMap, and Pod together:

```bash
cat <<'EOF' > time-check-all.yaml
# 1. Create the 'xfusion' namespace (safe to run even if it already exists)
apiVersion: v1
kind: Namespace
metadata:
  name: xfusion
---
# 2. Create the ConfigMap to store the sleep frequency
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-config
  namespace: xfusion
data:
  TIME_FREQ: "5"
---
# 3. Create the Pod that logs the date every 5 seconds
apiVersion: v1
kind: Pod
metadata:
  name: time-check
  namespace: xfusion
spec:
  containers:
  - name: time-check
    image: busybox:latest
    # Run the shell command and append (>>) the date output to the log file
    command: ["/bin/sh", "-c"]
    args: ["while true; do date; sleep $TIME_FREQ;done >> /opt/sysops/time/time-check.log"]
    # Pull the TIME_FREQ variable from the 'time-config' ConfigMap
    env:
    - name: TIME_FREQ
      valueFrom:
        configMapKeyRef:
          name: time-config
          key: TIME_FREQ
    # Mount the volume inside the container where the log file is written
    volumeMounts:
    - name: log-volume
      mountPath: /opt/sysops/time
  # Define the temporary emptyDir volume for storing logs
  volumes:
  - name: log-volume
    emptyDir: {}
EOF
```

### Step 2: Apply the Manifest
Deploy all resources to the cluster:

```bash
kubectl apply -f time-check-all.yaml
```

### Step 3: Verify the Deployment
Check that the Pod is running and confirm that timestamps are being written to the log file every 5 seconds:

```bash
# Check Pod status
kubectl get pod time-check -n xfusion

# Inspect the generated log file inside the container
kubectl exec -it time-check -n xfusion -- cat /opt/sysops/time/time-check.log
```

---

## 📝 Study Notes: Kubernetes ConfigMap

### What is a ConfigMap?
A **ConfigMap** is a Kubernetes API object used to store **non-confidential configuration data** in key-value pairs. It decouples environment-specific configuration from container images so applications remain portable.

### Why Use ConfigMaps?
- **Portability:** Reuse the exact same container image across `dev`, `staging`, and `prod` environments by attaching different ConfigMaps.
- **Easy Configuration Updates:** Modify application settings (such as `TIME_FREQ`) without rebuilding or pushing a new container image.

### Ways Pods Consume ConfigMaps
1. **Environment Variables (`env` / `envFrom`):** Inject ConfigMap keys as environment variables inside a container (used in this lab via `configMapKeyRef`).
2. **Command-Line Arguments:** Reference injected environment variables in the container's `command` or `args`.
3. **Mounted Volumes (`volumes`):** Mount a ConfigMap as a directory inside the container where each key becomes a file containing its value.

### ConfigMap vs. Secret

| Feature | ConfigMap | Secret |
| :--- | :--- | :--- |
| **Data Type** | Non-sensitive configuration data | Sensitive data (passwords, tokens, keys) |
| **Storage Format** | Plain text | Base64-encoded (with stricter RBAC/encryption options) |
| **Common Examples** | Ports, timeouts, feature flags, URLs | Database credentials, API keys, TLS certificates |
