# Kubernetes Secret and Deployment Example

This project demonstrates how to use a **Kubernetes Secret** to securely pass a database password to a backend application using a Deployment.

---

## Files Overview

### 1. `secret.yaml`

This file creates a **Kubernetes Secret** that stores a database password.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: RXhhbXBsZVBhc3N3b3JkMTIz
```

#### Explanation

* `kind: Secret` → Creates a Kubernetes Secret.
* `name: db-secret` → Name of the secret.
* `type: Opaque` → Generic secret type.
* `password` → Base64-encoded database password.

The value:

```
RXhhbXBsZVBhc3N3b3JkMTIz
```

is the Base64 encoding of:

```
ExamplePassword123
```

---

### 2. `deployment.yaml`

This file creates a **Deployment** that runs a backend container and injects the password from the Secret.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: nginx
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

#### Explanation

* `kind: Deployment` → Creates backend pods.
* `image: nginx` → Container image used.
* `env` → Defines environment variables.
* `DB_PASSWORD` → Injected from the Secret.

The container receives:

```
DB_PASSWORD=ExamplePassword123
```

---

## How It Works

1. The Secret stores the database password securely.
2. The Deployment references the Secret.
3. Kubernetes injects the password into the container as an environment variable.
4. The backend uses this password to connect to the database.

---

## What Happens If the Secret Is Missing?

If the `db-secret` is not created:

* The Deployment will fail to start properly.
* The container cannot access the database.
* The application may crash or return errors.

---

## Why Use Secrets?

### 1. Security

Passwords are not stored in:

* Source code
* Docker images
* Plain deployment files

### 2. Environment Flexibility

Different environments can use different passwords:

| Environment | Password |
| ----------- | -------- |
| Dev         | dev123   |
| Staging     | stage456 |
| Production  | prod789  |

---

## How to Deploy

### Step 1: Apply the Secret

```bash
kubectl apply -f secret.yaml
```

### Step 2: Apply the Deployment

```bash
kubectl apply -f deployment.yaml
```

---

## Verify the Deployment

Check pods:

```bash
kubectl get pods
```

Check environment variable inside the pod:

```bash
kubectl exec -it <pod-name> -- printenv DB_PASSWORD
```

---

## Summary

* Secret stores sensitive data.
* Deployment uses the Secret.
* Backend gets the password securely.
* Prevents exposure of credentials in code or configs.

---
