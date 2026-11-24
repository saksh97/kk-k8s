# Kubernetes Lab & Notes: Updating Deployments/Services & RBAC Fundamentals

This repository documents the step-by-step commands to update running Kubernetes resources without downtime/deletion, along with study notes on **Kubernetes RBAC (Role-Based Access Control)** and **User Management**.

---

## Part 1: Lab Solution — Updating Deployment & Service In-Place

### Objective
Modify the existing `nginx-service` and `nginx-deployment` on the `jump-host` without deleting the resources:
1. Update `nginx-service` NodePort from `30008` to `32165`.
2. Scale `nginx-deployment` from `1` to `5` replicas.
3. Update the container image in `nginx-deployment` from `nginx:1.18` to `nginx:latest`.

### Step-by-Step Commands

#### 1. Modify the Service NodePort
Patch `nginx-service` to replace `nodePort` with `32165`:
```bash
kubectl patch svc nginx-service --type='json' -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value": 32165}]'
```
*(Alternative: Run `kubectl edit svc nginx-service`, update `nodePort: 32165`, and save.)*

#### 2. Scale the Deployment Replicas
Increase the replica count of `nginx-deployment` to `5`:
```bash
kubectl scale deployment nginx-deployment --replicas=5
```

#### 3. Update the Container Image
Identify the container name inside the deployment and trigger a rolling update to `nginx:latest`:
```bash
# Check the container name
kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'

# Update the image (replace <container-name> with the output above, e.g., nginx)
kubectl set image deployment/nginx-deployment <container-name>=nginx:latest
```

#### 4. Verify the Changes
Confirm the service port, replica count, rollout status, and updated image:
```bash
kubectl get svc nginx-service
kubectl rollout status deployment/nginx-deployment
kubectl get deployment nginx-deployment
kubectl describe deployment nginx-deployment | grep Image
```

---

## Part 2: Learning Notes — Kubernetes RBAC & Users

### 1. What is RBAC?
**Role-Based Access Control (RBAC)** is a Kubernetes authorization mechanism (`rbac.authorization.k8s.io`) that restricts cluster access based on the roles assigned to users, groups, or service accounts. It enforces the **principle of least privilege** by defining *who* can perform *what actions (verbs)* on *which resources*.

#### Core RBAC Resources
| Resource | Scope | Purpose |
| :--- | :--- | :--- |
| **`Role`** | Namespace-scoped | Defines allowed actions (verbs) on resources within a single namespace. |
| **`ClusterRole`** | Cluster-wide | Defines permissions across all namespaces or on cluster-scoped resources (e.g., `nodes`, `persistentvolumes`). |
| **`RoleBinding`** | Namespace-scoped | Binds a `Role` (or `ClusterRole`) to a subject within a specific namespace. |
| **`ClusterRoleBinding`** | Cluster-wide | Binds a `ClusterRole` to a subject across the entire cluster. |

#### Example: Read-Only Pod Access (`Role` + `RoleBinding`)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: development
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### Testing RBAC Permissions
Verify whether a user or service account can perform an action:
```bash
kubectl auth can-i get pods --namespace=development --as=dev-user
```

---

### 2. Kubernetes Users: Where to Find Them?
Kubernetes **does not have a native `User` API resource** (so `kubectl get users` does not exist). Instead, Kubernetes distinguishes between two account types:

| Account Type | Managed By | Stored in Cluster? | How to Inspect |
| :--- | :--- | :--- | :--- |
| **Normal Users** (Humans) | External systems (X.509 client certificates, OIDC/LDAP, Cloud IAM) | **No** | Inspect RBAC bindings, `kubeconfig`, or Certificate Signing Requests (CSRs). |
| **ServiceAccounts** (Pods/Apps) | Kubernetes API (`kind: ServiceAccount`) | **Yes** | `kubectl get serviceaccounts -A` |

#### Useful Commands to Discover Users & Identities
```bash
# 1. Find human users referenced in ClusterRoleBindings
kubectl get clusterrolebindings -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.subjects[?(@.kind=="User")].name}{"\n"}{end}'

# 2. Find human users referenced in RoleBindings across all namespaces
kubectl get rolebindings -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.subjects[?(@.kind=="User")].name}{"\n"}{end}'

# 3. View users configured in your local kubeconfig
kubectl config get-users

# 4. List all ServiceAccounts across all namespaces
kubectl get serviceaccounts --all-namespaces

# 5. Check Certificate Signing Requests (for certificate-based users)
kubectl get csr
```
