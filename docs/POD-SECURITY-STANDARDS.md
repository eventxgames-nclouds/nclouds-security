# Pod Security Standards (PSS) for EKS

**Document:** SEC-1  
**Priority:** High  
**Target:** Week 6  
**Status:** Documentation Ready for Implementation

---

## Overview

This document defines the Pod Security Standards (PSS) implementation for the EventXGames EKS cluster. PSS is a Kubernetes-native security control that replaces the deprecated PodSecurityPolicy (PSP).

## PSS Levels

| Level | Description | Use Case |
|-------|-------------|----------|
| **Privileged** | Unrestricted policy | System workloads (monitoring agents) |
| **Baseline** | Minimally restrictive | Non-critical workloads |
| **Restricted** | Highly restrictive, best practices | Production application workloads |

**EventXGames Policy:** We use **Restricted** mode for all application namespaces.

---

## Namespace Configuration

### Application Namespace (eventxgames)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: eventxgames
  labels:
    # Enforce: Reject pods that violate the policy
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    
    # Audit: Log violations without blocking
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    
    # Warn: Show warnings to users deploying violating pods
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

### System Namespace (kube-system)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

---

## Restricted Policy Requirements

All pods in the `eventxgames` namespace must comply with:

### 1. Container Security

| Requirement | Value | Manifest Field |
|-------------|-------|----------------|
| Privileged | **false** | `securityContext.privileged` |
| Privilege Escalation | **false** | `securityContext.allowPrivilegeEscalation` |
| Run as Non-Root | **true** | `securityContext.runAsNonRoot` |
| Run as User | **>= 1000** | `securityContext.runAsUser` |
| Read-Only Root FS | **true** (recommended) | `securityContext.readOnlyRootFilesystem` |

### 2. Capabilities

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
    # Only add specific capabilities if absolutely required
    # add: []
```

### 3. Volume Types (Allowed)

- `configMap`
- `emptyDir`
- `projected`
- `secret`
- `downwardAPI`
- `persistentVolumeClaim`

**Not Allowed:** `hostPath`, `hostPID`, `hostIPC`, `hostNetwork`

### 4. Seccomp Profile

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

---

## Compliant Pod Template

### Frontend (Next.js)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: eventxgames
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: frontend
          image: eventxgames/frontend:latest
          ports:
            - containerPort: 3000
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: nextjs-cache
              mountPath: /app/.next/cache
      volumes:
        - name: tmp
          emptyDir: {}
        - name: nextjs-cache
          emptyDir: {}
```

### Backend API (Fastify)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: eventxgames
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: backend
          image: eventxgames/backend:latest
          ports:
            - containerPort: 3001
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          env:
            - name: NODE_ENV
              value: "production"
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

### AI Worker (Bedrock Integration)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-worker
  namespace: eventxgames
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ai-worker
  template:
    metadata:
      labels:
        app: ai-worker
    spec:
      serviceAccountName: ai-worker-sa  # For IRSA
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: worker
          image: eventxgames/ai-worker:latest
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /app/cache
      volumes:
        - name: tmp
          emptyDir: {}
        - name: cache
          emptyDir:
            sizeLimit: 1Gi
```

---

## Audit Process

### 1. Pre-Deployment Audit

Before deploying to the restricted namespace, run:

```bash
# Dry-run to check violations
kubectl apply -f deployment.yaml --dry-run=server -n eventxgames

# Check existing pods for violations
kubectl get pods -n eventxgames -o json | \
  kubectl-neat | \
  kubescape scan -
```

### 2. Runtime Audit

```bash
# View PSS violations in audit logs
kubectl logs -n kube-system -l component=kube-apiserver | grep "pod-security"

# List all pods with their security context
kubectl get pods -n eventxgames -o jsonpath='{range .items[*]}{.metadata.name}: runAsNonRoot={.spec.securityContext.runAsNonRoot}{"\n"}{end}'
```

### 3. Policy Validation Tools

| Tool | Purpose |
|------|---------|
| `kubescape` | Scan manifests for security issues |
| `kube-bench` | CIS benchmark compliance |
| `trivy` | Container image vulnerability scanning |
| `OPA Gatekeeper` | Custom policy enforcement |

---

## Common Violations and Fixes

### 1. Running as Root

**Violation:**
```yaml
securityContext:
  runAsUser: 0  # Root user
```

**Fix:**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
```

### 2. Privilege Escalation

**Violation:**
```yaml
securityContext:
  allowPrivilegeEscalation: true
```

**Fix:**
```yaml
securityContext:
  allowPrivilegeEscalation: false
```

### 3. Missing Seccomp Profile

**Violation:**
```yaml
# No seccompProfile specified
```

**Fix:**
```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

### 4. Excessive Capabilities

**Violation:**
```yaml
securityContext:
  capabilities:
    add:
      - NET_ADMIN
      - SYS_ADMIN
```

**Fix:**
```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

---

## Exemptions Process

For workloads that legitimately require elevated privileges:

1. **Document the requirement** - Explain why restricted mode cannot be used
2. **Minimize scope** - Request only necessary exemptions
3. **Approval required** - Security team must approve
4. **Use separate namespace** - Create a dedicated namespace with appropriate PSS level

### Exemption Namespace Example

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: eventxgames-system
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## Implementation Checklist

- [ ] PSS labels applied to `eventxgames` namespace
- [ ] All deployment manifests updated with security context
- [ ] Frontend pods compliant (runAsNonRoot, no privileged)
- [ ] Backend pods compliant
- [ ] AI worker pods compliant
- [ ] Audit logging enabled for violations
- [ ] No privileged containers in application namespace
- [ ] Documentation updated for developers

---

## References

- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [EKS Best Practices - Security](https://aws.github.io/aws-eks-best-practices/security/docs/)
- [AWS Well-Architected Framework - Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)

---

*Document created: 2026-04-21*  
*Part of: EVE-41 Well-Architected Remediation*
