# Kubernetes RBAC & Secrets Management

## Overview
Demonstrates Kubernetes security best practices through Role-Based Access Control (RBAC) implementation and secure secrets management. This project shows how to enforce least-privilege access and handle sensitive data in Kubernetes environments.

## Security Concepts Demonstrated

### 1. RBAC (Role-Based Access Control)
- **ServiceAccount:** Identity for pods and processes
- **Role:** Defines permissions within a namespace
- **RoleBinding:** Connects ServiceAccount to Role

### 2. Secrets Management
- Secure storage of sensitive data (passwords, tokens, keys)
- Base64 encoding for data protection
- Environment variable injection into pods
- Volume mounting for file-based secrets

## Architecture

```
ServiceAccount (WHO)
      ↓
RoleBinding (CONNECTS)
      ↓
Role (WHAT permissions)
      ↓
Pod uses ServiceAccount
      ↓
Access granted based on Role
```

## Project Structure

```
k8s-rbac-secrets/
├── rbac/
│   ├── serviceaccount.yaml    # Creates identity
│   ├── role.yaml               # Defines permissions
│   └── rolebinding.yaml        # Binds identity to role
└── secrets/
    ├── secret.yaml             # Stores sensitive data
    └── pod-using-secret.yaml   # Consumes the secret
```

## RBAC Implementation

### ServiceAccount
Creates an identity that pods can use:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader
  namespace: default
```

### Role
Defines what actions are allowed:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]  # Read-only access
```

**Permissions Explained:**
- `get`: Read a single pod
- `list`: List all pods
- `watch`: Watch for pod changes
- **NOT ALLOWED:** `create`, `update`, `delete` (enforces read-only)

### RoleBinding
Connects the ServiceAccount to the Role:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
subjects:
- kind: ServiceAccount
  name: pod-reader
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
```

## Secrets Management

### Creating Secrets
Two methods:

**Method 1: From literal values**
```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=supersecret
```

**Method 2: From YAML file**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=        # base64 encoded "admin"
  password: c3VwZXJzZWNyZXQ=  # base64 encoded "supersecret"
```

### Using Secrets in Pods

**As Environment Variables:**
```yaml
env:
- name: DB_USERNAME
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: username
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

**As Volume Mounts:**
```yaml
volumes:
- name: secret-volume
  secret:
    secretName: db-secret
volumeMounts:
- name: secret-volume
  mountPath: /etc/secrets
  readOnly: true
```

## How to Deploy

### Prerequisites
- Kubernetes cluster (Minikube, kind, or cloud)
- kubectl configured
- Namespace created (or use default)

### Step 1: Deploy RBAC
```bash
# Create ServiceAccount
kubectl apply -f rbac/serviceaccount.yaml

# Create Role
kubectl apply -f rbac/role.yaml

# Create RoleBinding
kubectl apply -f rbac/rolebinding.yaml

# Verify
kubectl get serviceaccounts
kubectl get roles
kubectl get rolebindings
```

### Step 2: Deploy Secrets
```bash
# Create secret
kubectl apply -f secrets/secret.yaml

# Verify (don't show values)
kubectl get secrets

# View decoded secret (for testing only)
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

### Step 3: Deploy Pod with Secret
```bash
kubectl apply -f secrets/pod-using-secret.yaml

# Verify pod is running
kubectl get pods

# Check if secret is accessible inside pod
kubectl exec -it <pod-name> -- env | grep DB_
```

## Testing RBAC Permissions

### Test Read Access (Should Work)
```bash
# Use the ServiceAccount to list pods
kubectl auth can-i list pods --as=system:serviceaccount:default:pod-reader
# Output: yes

kubectl auth can-i get pods --as=system:serviceaccount:default:pod-reader
# Output: yes
```

### Test Write Access (Should Fail)
```bash
# Try to create/delete (should be denied)
kubectl auth can-i create pods --as=system:serviceaccount:default:pod-reader
# Output: no

kubectl auth can-i delete pods --as=system:serviceaccount:default:pod-reader
# Output: no
```

## Security Best Practices Implemented

### RBAC
✅ **Least Privilege:** Only read permissions granted, no write access
✅ **Namespace Isolation:** Role limited to specific namespace
✅ **Explicit Permissions:** Only necessary resources and verbs defined
✅ **ServiceAccount per App:** Each application has its own identity

### Secrets
✅ **No Plain Text:** Passwords base64 encoded
✅ **No Hardcoded Credentials:** Secrets separate from application code
✅ **Environment Injection:** Secrets loaded at runtime, not in image
✅ **Read-Only Mounts:** Secret volumes mounted read-only

## What I Learned

### Kubernetes Security Model
- How RBAC controls access to Kubernetes API
- Difference between Role (namespace) and ClusterRole (cluster-wide)
- How ServiceAccounts work for pods
- Binding multiple ServiceAccounts to same Role

### Secrets Management
- Why base64 encoding ≠ encryption (just obfuscation)
- Different ways to inject secrets into pods
- Secret lifecycle management
- When to use Secrets vs ConfigMaps

### Best Practices
- Always use RBAC in production
- Never commit secrets to Git
- Use external secret managers (AWS Secrets Manager, HashiCorp Vault) for production
- Rotate secrets regularly
- Audit access logs

## Common Issues & Solutions

### Issue 1: "Forbidden" Error
```
Error: pods is forbidden: User "system:serviceaccount:default:pod-reader" 
cannot list resource "pods"
```
**Solution:** Check RoleBinding is correct and applied

### Issue 2: Secret Not Found
```
Error: couldn't find key password in Secret default/db-secret
```
**Solution:** Verify secret key name matches in both Secret and Pod definition

### Issue 3: Base64 Decoding Issues
**Solution:** Use proper encoding:
```bash
echo -n "mypassword" | base64  # -n flag prevents newline
```

## Real-World Use Cases

### Database Access
- Store database credentials as secrets
- Mount into application pods
- Rotate without redeploying application

### API Keys
- Store third-party API keys securely
- Inject as environment variables
- Different secrets for dev/staging/prod

### TLS Certificates
- Store SSL/TLS certificates as secrets
- Mount into Ingress or service mesh
- Automatic rotation support

## Comparison: ConfigMap vs Secret

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Use Case** | Non-sensitive config | Passwords, tokens, keys |
| **Encoding** | Plain text | Base64 encoded |
| **Size Limit** | 1MB | 1MB |
| **Encrypted at Rest** | No | Optional (etcd encryption) |
| **Example** | App settings, URLs | DB passwords, API keys |

## Advanced Topics (Future Learning)

- **ClusterRole & ClusterRoleBinding:** Cluster-wide permissions
- **Pod Security Policies:** Restrict what pods can do
- **Network Policies:** Control pod-to-pod communication
- **External Secrets Operator:** Sync from AWS/GCP/Azure
- **Sealed Secrets:** Encrypt secrets in Git
- **HashiCorp Vault Integration:** Dynamic secret generation

## Related Projects
- [DevOps End-to-End Project](https://github.com/Kumaranvelan/devops-end-to-end-project) - Uses RBAC + Secrets
- [Jenkins Docker Demo](https://github.com/Kumaranvelan/jenkins_docker_demo) - Credential management
- [K8s Networking Demo](https://github.com/Kumaranvelan/k8s-networking-service-types-demo) - Service security

## References
- [Kubernetes RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
- [CNCF Security Best Practices](https://www.cncf.io/blog/2020/11/18/kubernetes-security-best-practices/)

## Contact
**Kumaravelan Subramani**
- GitHub: [@Kumaranvelan](https://github.com/Kumaranvelan)
- LinkedIn: [kumaravelan-subramani](https://linkedin.com/in/kumaravelan-subramani-a1399b253)

---

*This project demonstrates Kubernetes security fundamentals essential for production deployments.*
