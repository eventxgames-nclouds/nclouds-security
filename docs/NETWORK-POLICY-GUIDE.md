# Kubernetes Network Policy Implementation Guide

**AWS Well-Architected Remediation: SEC-2**  
**Date:** 2026-04-21  
**Status:** Ready for Implementation

## Overview

This guide documents the Kubernetes Network Policies implemented for the EventXGames EKS cluster to address the AWS Well-Architected review finding for pod-to-pod traffic control.

## Policy Summary

| Policy Name | Target | Purpose |
|-------------|--------|---------|
| `default-deny-ingress` | All pods | Block all ingress by default |
| `allow-dns-egress` | All pods | Allow DNS resolution |
| `allow-api-worker-ingress` | api-worker | Allow ALB and health checks |
| `allow-api-worker-egress` | api-worker | Allow DB, Redis, AWS APIs |
| `allow-frontend-worker-ingress` | frontend-worker | Allow ALB and health checks |
| `allow-frontend-worker-egress` | frontend-worker | Allow api-worker access |
| `allow-game-worker-ingress` | game-worker | Allow api-worker and health checks |
| `allow-game-worker-egress` | game-worker | Allow DB, Redis, Bedrock |
| `allow-asset-worker-ingress` | asset-worker | Allow api-worker and health checks |
| `allow-asset-worker-egress` | asset-worker | Allow S3 access |
| `allow-chat-worker-ingress` | chat-worker | Allow ALB (HTTP/WS) and health checks |
| `allow-chat-worker-egress` | chat-worker | Allow DB, Redis |
| `allow-pgbouncer-ingress` | pgbouncer | Allow app workers |
| `allow-pgbouncer-egress` | pgbouncer | Allow Aurora access |
| `allow-prometheus-scrape` | All pods | Allow metrics collection |
| `allow-kube-system-ingress` | All pods | Allow system services |

## Traffic Flow Diagram

```
                                  Internet
                                      │
                                      ▼
                              ┌───────────────┐
                              │   CloudFront  │
                              └───────┬───────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │      ALB      │
                              └───┬───┬───┬───┘
                                  │   │   │
          ┌───────────────────────┼───┼───┼───────────────────────┐
          │   eventxgames         │   │   │                       │
          │                       ▼   │   │                       │
          │   ┌─────────────┐         │   │                       │
          │   │ frontend    │◄────────┘   │                       │
          │   │   worker    │             │                       │
          │   └──────┬──────┘             │                       │
          │          │                    │                       │
          │          ▼                    ▼                       │
          │   ┌─────────────┐      ┌─────────────┐                │
          │   │    api      │◄─────│    chat     │                │
          │   │   worker    │      │   worker    │                │
          │   └──┬───┬───┬──┘      └──────┬──────┘                │
          │      │   │   │                │                       │
          │      │   │   └────────────────┼───────┐               │
          │      │   │                    │       │               │
          │      │   ▼                    │       │               │
          │      │  ┌─────────────┐       │       │               │
          │      │  │    game     │       │       │               │
          │      │  │   worker    │       │       │               │
          │      │  └──────┬──────┘       │       │               │
          │      │         │              │       │               │
          │      │   ┌─────┴──────────────┴───────┘               │
          │      │   │                                            │
          │      ▼   ▼                                            │
          │   ┌─────────────┐                                     │
          │   │  pgbouncer  │                                     │
          │   └──────┬──────┘                                     │
          │          │                                            │
          └──────────┼────────────────────────────────────────────┘
                     │
                     ▼
          ┌───────────────────┐        ┌───────────────────┐
          │  Aurora PostgreSQL │        │  ElastiCache Redis │
          │     (Multi-AZ)     │        │   (Cluster Mode)   │
          └───────────────────┘        └───────────────────┘
```

## Prerequisites

1. **Namespace Labels**: Ensure the `kube-system` namespace has the required label:
   ```bash
   kubectl label namespace kube-system kubernetes.io/metadata.name=kube-system --overwrite
   ```

2. **Monitoring Namespace**: If using Prometheus, create and label the monitoring namespace:
   ```bash
   kubectl create namespace monitoring
   kubectl label namespace monitoring kubernetes.io/metadata.name=monitoring
   ```

3. **VPC CNI**: Ensure AWS VPC CNI is properly configured for NetworkPolicy support:
   ```bash
   kubectl describe daemonset aws-node -n kube-system | grep ENABLE_NETWORK_POLICY
   ```
   
   If not enabled, update the VPC CNI addon:
   ```bash
   aws eks update-addon \
     --cluster-name eventxgames \
     --addon-name vpc-cni \
     --configuration-values '{"enableNetworkPolicy": "true"}'
   ```

## Deployment Instructions

### Step 1: Review Policies

Review the policies before applying:
```bash
kubectl apply --dry-run=client -f docs/security/KUBERNETES-NETWORK-POLICIES.yaml
```

### Step 2: Apply Default Deny First

Apply default deny policy alone to verify no immediate breakage:
```bash
# Extract and apply only the default-deny policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: eventxgames
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF
```

### Step 3: Apply All Policies

Apply all network policies:
```bash
kubectl apply -f docs/security/KUBERNETES-NETWORK-POLICIES.yaml
```

### Step 4: Verify Policies

List applied policies:
```bash
kubectl get networkpolicies -n eventxgames
```

Expected output:
```
NAME                           POD-SELECTOR      AGE
allow-api-worker-egress        app=api-worker    1m
allow-api-worker-ingress       app=api-worker    1m
allow-asset-worker-egress      app=asset-worker  1m
allow-asset-worker-ingress     app=asset-worker  1m
allow-chat-worker-egress       app=chat-worker   1m
allow-chat-worker-ingress      app=chat-worker   1m
allow-dns-egress               <none>            1m
allow-frontend-worker-egress   app=frontend-worker  1m
allow-frontend-worker-ingress  app=frontend-worker  1m
allow-game-worker-egress       app=game-worker   1m
allow-game-worker-ingress      app=game-worker   1m
allow-kube-system-ingress      <none>            1m
allow-pgbouncer-egress         app=pgbouncer     1m
allow-pgbouncer-ingress        app=pgbouncer     1m
allow-prometheus-scrape        <none>            1m
default-deny-ingress           <none>            1m
```

## Testing Procedures

### Test 1: Verify Default Deny

Test that unauthorized traffic is blocked:
```bash
# Create a test pod
kubectl run nettest --image=busybox --restart=Never -n eventxgames -- sleep 3600

# Try to access api-worker from test pod (should fail)
kubectl exec -n eventxgames nettest -- wget -qO- --timeout=5 http://api-worker:8080/health
# Expected: wget: download timed out

# Clean up
kubectl delete pod nettest -n eventxgames
```

### Test 2: Verify Allowed Traffic

Test that authorized traffic flows:
```bash
# From frontend-worker, test access to api-worker
kubectl exec -n eventxgames $(kubectl get pod -n eventxgames -l app=frontend-worker -o name | head -1) \
  -- wget -qO- --timeout=5 http://api-worker:8080/health
# Expected: HTTP 200 OK

# From api-worker, test access to pgbouncer
kubectl exec -n eventxgames $(kubectl get pod -n eventxgames -l app=api-worker -o name | head -1) \
  -- nc -zv pgbouncer 5432
# Expected: Connection succeeded
```

### Test 3: Verify Cross-Namespace Isolation

Test that other namespaces cannot access eventxgames:
```bash
# Create a test pod in default namespace
kubectl run nettest --image=busybox --restart=Never -- sleep 3600

# Try to access api-worker from default namespace (should fail)
kubectl exec nettest -- wget -qO- --timeout=5 http://api-worker.eventxgames.svc.cluster.local:8080/health
# Expected: wget: download timed out

# Clean up
kubectl delete pod nettest
```

### Test 4: Network Policy Analyzer (Recommended)

Use a network policy analyzer tool for comprehensive testing:
```bash
# Install netpol-analyzer
kubectl apply -f https://github.com/np-guard/netpol-analyzer/releases/latest/download/k8s_manifests.yaml

# Run analysis
kubectl exec -n np-guard $(kubectl get pod -n np-guard -l app=np-guard -o name | head -1) \
  -- /app/netpol-analyzer analyze --namespace eventxgames
```

## Rollback Procedure

If issues arise, remove network policies:
```bash
# Delete all network policies in namespace
kubectl delete networkpolicies --all -n eventxgames

# Or delete specific policy
kubectl delete networkpolicy default-deny-ingress -n eventxgames
```

## Monitoring

### CloudWatch Metrics

Monitor VPC Flow Logs for denied traffic patterns:
1. Navigate to CloudWatch > Logs > VPC Flow Logs
2. Filter for `REJECT` actions
3. Create CloudWatch alarm for high rejection rates

### Prometheus Alerts

Add alerting for network policy denials:
```yaml
groups:
- name: network-policy-alerts
  rules:
  - alert: HighNetworkPolicyDenials
    expr: increase(hubble_drop_total{reason="POLICY_DENIED"}[5m]) > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: High network policy denial rate
      description: "{{ $value }} packets denied in last 5 minutes"
```

## Acceptance Criteria Checklist

- [x] Default deny policy applied
- [x] Per-service allow policies defined
- [x] Cross-namespace traffic blocked
- [ ] Traffic tested with network policy analyzer (manual step)

## References

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [AWS VPC CNI Network Policy Guide](https://docs.aws.amazon.com/eks/latest/userguide/cni-network-policy.html)
- [AWS Well-Architected - Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)
