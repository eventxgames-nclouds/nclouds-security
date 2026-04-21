# EventXGames Security Documentation

This repository contains security audit plans, compliance checklists, and IAM policy documentation for the EventXGames AWS migration.

## Contents

- [COMPLIANCE_CHECKLIST.md](./COMPLIANCE_CHECKLIST.md) - Security compliance checklist for AWS migration
- [SECURITY_AUDIT_PLAN.md](./SECURITY_AUDIT_PLAN.md) - Comprehensive security audit plan

## Security Overview

### Compliance Frameworks

| Framework | Status | Notes |
|-----------|--------|-------|
| SOC2 Type II | In Progress | Evidence collection automated |
| GDPR | In Progress | Data residency in eu-west-1 for EU users |
| AWS Well-Architected | Planned | Security pillar review |

### Key Security Controls

#### Identity & Access Management
- Least privilege IAM roles
- MFA enforcement for all users
- IRSA (IAM Roles for Service Accounts) for EKS
- Session tags for cross-account access

#### Network Security
- Multi-AZ VPC with public/private subnets
- Security groups per service tier
- VPC endpoints for AWS services
- WAF on ALB with managed rules

#### Data Protection
- Encryption at rest (CMK for Aurora, S3, EBS)
- TLS 1.2+ for all traffic
- Secrets Manager with automatic rotation
- Key rotation enabled

#### Monitoring & Detection
- GuardDuty enabled in all regions
- Security Hub with CIS benchmarks
- CloudWatch alarms for security events
- CloudTrail for API logging

## Security Checklist Progress

### Pre-Migration
- [x] Security architecture approved
- [x] IAM design reviewed
- [x] Network design reviewed
- [ ] Compliance requirements mapped
- [ ] Pen test authorization obtained

### Go-Live Readiness
- [ ] All logging enabled
- [ ] All encryption configured
- [ ] Security monitoring active
- [ ] Incident response tested
- [ ] WAF in BLOCK mode

## Quick Links

- [AWS Architecture](https://github.com/eventxgames-nclouds/nclouds-aws-architecture)
- [Infrastructure Code](https://github.com/eventxgames-nclouds/nclouds-infrastructure)
- [Main Project Index](https://github.com/eventxgames-nclouds/nclouds-overview)
