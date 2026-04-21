# Security Audit Plan: Azure to AWS Migration
## EventXGames Platform

**Document Version:** 1.0  
**Date:** 2026-04-16  
**Author:** Security Engineer Agent  
**Classification:** Internal - Confidential

---

## Executive Summary

This document outlines the comprehensive security audit plan for migrating EventXGames from Azure VPS to AWS infrastructure (EKS, Aurora, Bedrock). The plan addresses identity management, network security, data protection, compliance requirements, and security monitoring.

---

## 1. IAM Policies and Roles Design

### 1.1 Principle of Least Privilege

| Role Category | Scope | Permissions |
|---------------|-------|-------------|
| EKS Cluster Admin | Cluster-wide | `eks:*`, limited to specific clusters |
| Application Pod Role | Namespace-scoped | Read Secrets Manager, write CloudWatch logs |
| CI/CD Pipeline | Deploy-only | `ecr:PushImage`, `eks:UpdateNodegroup` |
| Aurora Admin | Database | `rds:*` on Aurora cluster only |
| Bedrock Consumer | ML inference | `bedrock:InvokeModel` for specific models |

### 1.2 IAM Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["eks:DescribeCluster", "eks:ListClusters"],
      "Resource": "arn:aws:eks:*:ACCOUNT_ID:cluster/eventxgames-*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
        }
      }
    }
  ]
}
```

### 1.3 Service Account Mappings (IRSA)

| Kubernetes Service Account | IAM Role | Purpose |
|---------------------------|----------|---------|
| `app-backend` | `EventXGames-Backend-Role` | S3 access, Secrets Manager |
| `app-worker` | `EventXGames-Worker-Role` | SQS, Bedrock invocation |
| `monitoring` | `EventXGames-Monitoring-Role` | CloudWatch, X-Ray |

### 1.4 Audit Actions
- [ ] Enable CloudTrail for all IAM events
- [ ] Configure IAM Access Analyzer
- [ ] Implement session tags for cross-account access
- [ ] Set up MFA enforcement for console access
- [ ] Define password policy (min 14 chars, 90-day rotation)

---

## 2. WAF Configuration for API Protection

### 2.1 AWS WAF Rules

| Rule Name | Type | Action | Priority |
|-----------|------|--------|----------|
| RateLimitRule | Rate-based | Block @ 2000 req/5min | 1 |
| SQLInjectionRule | Managed | Block | 2 |
| XSSRule | Managed | Block | 3 |
| GeoBlockRule | Custom | Block non-allowed countries | 4 |
| BotControlRule | Managed | Challenge | 5 |

### 2.2 Rule Group Configuration

```yaml
WebACL:
  Name: eventxgames-api-waf
  Rules:
    - Name: AWSManagedRulesCommonRuleSet
      Priority: 1
      OverrideAction: None
    - Name: AWSManagedRulesSQLiRuleSet
      Priority: 2
      OverrideAction: None
    - Name: AWSManagedRulesKnownBadInputsRuleSet
      Priority: 3
      OverrideAction: None
    - Name: CustomRateLimiting
      Priority: 10
      Statement:
        RateBasedStatement:
          Limit: 2000
          AggregateKeyType: IP
```

### 2.3 API Gateway Integration
- Attach WAF WebACL to API Gateway stages
- Enable detailed CloudWatch logging
- Configure real-time metrics for blocked requests

### 2.4 Audit Actions
- [ ] Deploy WAF in COUNT mode initially (2-week baseline)
- [ ] Review false positives before switching to BLOCK
- [ ] Set up SNS alerts for high block rates
- [ ] Document exception rules with business justification

---

## 3. Secrets Management (AWS Secrets Manager)

### 3.1 Secret Categories

| Category | Rotation Period | Encryption |
|----------|----------------|------------|
| Database credentials | 30 days | CMK |
| API keys (internal) | 90 days | CMK |
| API keys (third-party) | Manual | CMK |
| TLS certificates | Cert Manager auto | ACM |

### 3.2 Secret Naming Convention

```
/eventxgames/{environment}/{service}/{secret-name}
Example: /eventxgames/prod/backend/aurora-primary-creds
```

### 3.3 Access Control

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:*:*:secret:/eventxgames/prod/backend/*",
      "Condition": {
        "StringEquals": {
          "secretsmanager:VersionStage": "AWSCURRENT"
        }
      }
    }
  ]
}
```

### 3.4 Integration with EKS
- Use External Secrets Operator (ESO) for Kubernetes
- Map AWS Secrets Manager to Kubernetes Secrets
- Implement secret rotation Lambda functions

### 3.5 Audit Actions
- [ ] Enable automatic rotation for all database credentials
- [ ] Configure CloudTrail for `secretsmanager:GetSecretValue` events
- [ ] Set up alerts for failed secret access attempts
- [ ] Document emergency secret rotation procedure

---

## 4. Network Security (VPC, Security Groups)

### 4.1 VPC Architecture

```
Region: us-east-1 (primary), eu-west-1 (DR)

VPC CIDR: 10.0.0.0/16
├── Public Subnets (10.0.0.0/20)
│   ├── us-east-1a: 10.0.0.0/22 (ALB, NAT Gateway)
│   ├── us-east-1b: 10.0.4.0/22
│   └── us-east-1c: 10.0.8.0/22
├── Private Subnets - Application (10.0.16.0/20)
│   ├── us-east-1a: 10.0.16.0/22 (EKS nodes)
│   ├── us-east-1b: 10.0.20.0/22
│   └── us-east-1c: 10.0.24.0/22
└── Private Subnets - Data (10.0.32.0/20)
    ├── us-east-1a: 10.0.32.0/22 (Aurora, ElastiCache)
    ├── us-east-1b: 10.0.36.0/22
    └── us-east-1c: 10.0.40.0/22
```

### 4.2 Security Groups

| Security Group | Inbound | Outbound |
|----------------|---------|----------|
| `sg-alb` | 443 from 0.0.0.0/0 | All to `sg-eks` |
| `sg-eks` | All from `sg-alb`, 443 from VPC | 443 to VPC endpoints, 5432 to `sg-aurora` |
| `sg-aurora` | 5432 from `sg-eks` only | None |
| `sg-bedrock-endpoint` | 443 from `sg-eks` | None |

### 4.3 VPC Endpoints (PrivateLink)

| Service | Type | Purpose |
|---------|------|---------|
| com.amazonaws.region.secretsmanager | Interface | Secrets access without IGW |
| com.amazonaws.region.ecr.api | Interface | Container registry |
| com.amazonaws.region.ecr.dkr | Interface | Container image pull |
| com.amazonaws.region.s3 | Gateway | S3 access |
| com.amazonaws.region.bedrock-runtime | Interface | ML inference |

### 4.4 Network ACLs

| Rule | Direction | Protocol | Port | Source/Dest | Action |
|------|-----------|----------|------|-------------|--------|
| 100 | Inbound | TCP | 443 | 0.0.0.0/0 | Allow |
| 110 | Inbound | TCP | 1024-65535 | VPC CIDR | Allow |
| 200 | Inbound | ALL | ALL | 0.0.0.0/0 | Deny |

### 4.5 Audit Actions
- [ ] Enable VPC Flow Logs to CloudWatch
- [ ] Configure VPC Traffic Mirroring for forensics capability
- [ ] Implement AWS Network Firewall for egress filtering
- [ ] Document all allowed cross-AZ traffic patterns

---

## 5. SOC2/GDPR Compliance Checklist

### 5.1 SOC2 Type II Controls

| Control ID | Category | Requirement | Implementation |
|------------|----------|-------------|----------------|
| CC6.1 | Logical Access | Access controls implemented | IAM + RBAC |
| CC6.2 | Logical Access | User provisioning process | AWS SSO + IDP |
| CC6.3 | Logical Access | Access removal process | Automated offboarding |
| CC6.6 | Logical Access | MFA required | Enforced via IAM policy |
| CC6.7 | Logical Access | System access logging | CloudTrail + CloudWatch |
| CC7.1 | System Operations | Change management | GitOps + approval workflow |
| CC7.2 | System Operations | Infrastructure monitoring | CloudWatch + X-Ray |
| CC8.1 | Risk Mitigation | Incident response plan | Documented, tested quarterly |

### 5.2 GDPR Requirements

| Article | Requirement | Implementation |
|---------|-------------|----------------|
| Art. 25 | Data protection by design | Encryption at rest/transit |
| Art. 30 | Records of processing | Data flow documentation |
| Art. 32 | Security of processing | This audit plan |
| Art. 33 | Breach notification | SNS + PagerDuty (72hr SLA) |
| Art. 35 | DPIA | Completed for Bedrock usage |

### 5.3 Data Residency

| Data Type | Storage Location | Encryption | Retention |
|-----------|------------------|------------|-----------|
| User PII | Aurora (eu-west-1) | AES-256 CMK | 7 years |
| Game telemetry | S3 (multi-region) | AES-256 CMK | 2 years |
| Logs | CloudWatch (same region) | Default | 90 days |
| Backups | S3 Glacier | AES-256 CMK | 7 years |

### 5.4 Audit Actions
- [ ] Complete data mapping inventory
- [ ] Implement data subject access request (DSAR) automation
- [ ] Configure AWS Macie for PII detection
- [ ] Schedule annual third-party SOC2 audit
- [ ] Document lawful basis for each data processing activity

---

## 6. Penetration Testing Plan

### 6.1 Scope

| Target | Type | Methodology |
|--------|------|-------------|
| API Gateway endpoints | Black-box | OWASP API Security Top 10 |
| EKS cluster | Gray-box | CIS Kubernetes Benchmark |
| Aurora database | White-box | SQL injection, privilege escalation |
| IAM configuration | White-box | Privilege escalation, policy analysis |

### 6.2 Schedule

| Phase | Duration | Activities |
|-------|----------|------------|
| Reconnaissance | Week 1 | Asset discovery, OSINT |
| Vulnerability Assessment | Week 2 | Automated scanning (Nessus, Prowler) |
| Exploitation | Week 3-4 | Manual testing, exploit validation |
| Reporting | Week 5 | Findings documentation, risk scoring |

### 6.3 AWS Authorization
- Submit AWS Penetration Testing Request form
- Allowed services: EC2, EKS, RDS, Lambda, API Gateway
- Prohibited: DDoS, zone walking, DNS hijacking

### 6.4 Testing Tools

| Tool | Purpose | License |
|------|---------|---------|
| Prowler | AWS security posture | Open source |
| ScoutSuite | Cloud security audit | Open source |
| kube-hunter | Kubernetes security | Open source |
| Burp Suite Pro | API testing | Commercial |
| Nuclei | Vulnerability scanning | Open source |

### 6.5 Audit Actions
- [ ] Submit AWS authorization request (7-day lead time)
- [ ] Set up isolated testing environment
- [ ] Configure alerting exclusions for pen test IPs
- [ ] Establish emergency rollback procedures
- [ ] Define severity classification matrix

---

## 7. Security Monitoring Setup

### 7.1 Detection Architecture

```
┌─────────────┐     ┌───────────────┐     ┌──────────────┐
│ CloudTrail  │────►│ CloudWatch    │────►│ Security Hub │
│ VPC Logs    │     │ Log Groups    │     │              │
│ EKS Logs    │     └───────┬───────┘     └──────┬───────┘
└─────────────┘             │                    │
                            ▼                    ▼
                    ┌───────────────┐     ┌──────────────┐
                    │ EventBridge   │────►│ SNS/PagerDuty│
                    │ Rules         │     │ Alerts       │
                    └───────────────┘     └──────────────┘
```

### 7.2 GuardDuty Findings

| Finding Type | Severity | Response |
|--------------|----------|----------|
| UnauthorizedAccess:IAMUser | High | Auto-disable key, page oncall |
| Recon:EC2/PortProbeUnprotected | Medium | Log, investigate |
| CryptoCurrency:EC2/BitcoinTool | High | Isolate instance, forensics |

### 7.3 CloudWatch Alarms

| Metric | Threshold | Action |
|--------|-----------|--------|
| Failed console logins | >5/min | SNS alert |
| Root account usage | >0 | PagerDuty + Slack |
| Unauthorized API calls | >10/min | SNS alert |
| Security group changes | Any | Slack notification |

### 7.4 SIEM Integration
- Export logs to central SIEM (Splunk/Elasticsearch)
- Correlation rules for multi-stage attacks
- 90-day hot retention, 1-year cold storage

### 7.5 Audit Actions
- [ ] Enable GuardDuty in all regions
- [ ] Configure Security Hub with CIS AWS Foundations Benchmark
- [ ] Set up cross-account CloudTrail aggregation
- [ ] Implement automated remediation for critical findings
- [ ] Document incident response runbooks

---

## Appendix A: Compliance Checklist Summary

### Pre-Migration
- [ ] IAM roles and policies documented and reviewed
- [ ] VPC architecture approved by security team
- [ ] WAF rules tested in staging
- [ ] Secrets rotation procedures validated
- [ ] Penetration test authorization obtained

### During Migration
- [ ] Enable all logging before go-live
- [ ] Verify encryption at rest for all data stores
- [ ] Confirm VPC endpoints functioning
- [ ] Test security group rules with traffic analysis
- [ ] Validate IRSA mappings in EKS

### Post-Migration
- [ ] Execute penetration test
- [ ] Complete SOC2 evidence collection
- [ ] Verify GDPR data flows documented
- [ ] Conduct tabletop incident response exercise
- [ ] Schedule recurring security reviews (quarterly)

---

## Appendix B: Risk Register

| Risk ID | Description | Likelihood | Impact | Mitigation |
|---------|-------------|------------|--------|------------|
| R-001 | Misconfigured S3 bucket exposure | Medium | High | S3 Block Public Access, Macie |
| R-002 | IAM privilege escalation | Medium | Critical | IAM Access Analyzer, least privilege |
| R-003 | Container escape in EKS | Low | Critical | Pod Security Standards, Falco |
| R-004 | Data breach via SQL injection | Low | High | WAF, parameterized queries |
| R-005 | Insider threat | Low | High | CloudTrail, UEBA |

---

## Document Approval

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Security Engineer | Agent | 2026-04-16 | Pending |
| CTO | | | |
| Compliance Officer | | | |
