# AWS Migration Security Compliance Checklist
## EventXGames Platform

**Version:** 1.0  
**Date:** 2026-04-16

---

## Identity & Access Management (IAM)

### Policies
- [ ] Implement least privilege for all IAM roles
- [ ] Enable MFA for all human users
- [ ] Configure password policy (14+ chars, 90-day rotation)
- [ ] Set up IAM Access Analyzer
- [ ] Enable CloudTrail for IAM events
- [ ] Define service control policies (SCPs) for guardrails
- [ ] Implement session tags for cross-account access

### EKS-Specific
- [ ] Configure IRSA (IAM Roles for Service Accounts)
- [ ] Map Kubernetes service accounts to IAM roles
- [ ] Implement Pod Security Standards
- [ ] Enable EKS audit logging

---

## Network Security

### VPC Configuration
- [ ] Design multi-AZ VPC with public/private subnets
- [ ] Configure NAT Gateways for private subnet egress
- [ ] Enable VPC Flow Logs
- [ ] Set up VPC endpoints for AWS services (S3, ECR, Secrets Manager)
- [ ] Implement network ACLs as defense-in-depth

### Security Groups
- [ ] Define security groups per service tier
- [ ] Restrict database access to application subnets only
- [ ] Block all inbound by default, allow specific ports
- [ ] Document all security group rules

### Load Balancing
- [ ] Configure ALB with TLS 1.2+ only
- [ ] Enable WAF on ALB
- [ ] Set up DDoS protection (Shield Standard)

---

## Web Application Firewall (WAF)

### Rule Configuration
- [ ] Deploy AWS Managed Rules (Common, SQLi, XSS)
- [ ] Configure rate limiting (2000 req/5min per IP)
- [ ] Set up geo-blocking for unauthorized regions
- [ ] Enable Bot Control managed rule group
- [ ] Create custom rules for application-specific threats

### Monitoring
- [ ] Deploy in COUNT mode for 2-week baseline
- [ ] Review logs for false positives
- [ ] Configure SNS alerts for high block rates
- [ ] Document exception rules with justification

---

## Secrets Management

### AWS Secrets Manager
- [ ] Define secret naming convention
- [ ] Enable automatic rotation for database credentials (30 days)
- [ ] Set up CMK encryption for secrets
- [ ] Configure resource policies for access control

### EKS Integration
- [ ] Deploy External Secrets Operator
- [ ] Map AWS secrets to Kubernetes secrets
- [ ] Test secret rotation in non-prod

### Audit
- [ ] Enable CloudTrail for GetSecretValue events
- [ ] Alert on failed secret access attempts
- [ ] Document emergency rotation procedure

---

## Data Protection

### Encryption at Rest
- [ ] Aurora: Enable encryption with CMK
- [ ] S3: Enable default encryption with CMK
- [ ] EBS: Enable encryption for EKS node volumes
- [ ] Backups: Encrypt with separate CMK

### Encryption in Transit
- [ ] TLS 1.2+ for all external endpoints
- [ ] TLS between EKS pods (service mesh or mTLS)
- [ ] SSL connections to Aurora
- [ ] HTTPS for all VPC endpoint traffic

### Key Management
- [ ] Create CMKs per environment (prod/staging/dev)
- [ ] Enable key rotation
- [ ] Define key policies with least privilege
- [ ] Document key recovery procedures

---

## Monitoring & Detection

### AWS GuardDuty
- [ ] Enable in all regions
- [ ] Configure findings export to Security Hub
- [ ] Set up SNS notifications for High/Critical findings
- [ ] Define auto-remediation for common threats

### AWS Security Hub
- [ ] Enable CIS AWS Foundations Benchmark
- [ ] Enable AWS Foundational Security Best Practices
- [ ] Configure cross-account aggregation
- [ ] Schedule weekly finding reviews

### CloudWatch
- [ ] Create alarms for security events
- [ ] Set up dashboard for security metrics
- [ ] Configure log retention (90 days minimum)
- [ ] Enable anomaly detection

### Incident Response
- [ ] Define severity classification matrix
- [ ] Document runbooks for common incidents
- [ ] Set up PagerDuty integration
- [ ] Schedule quarterly tabletop exercises

---

## SOC2 Type II Requirements

### Access Control (CC6)
- [ ] CC6.1: Logical access controls implemented
- [ ] CC6.2: User provisioning process documented
- [ ] CC6.3: Access removal process automated
- [ ] CC6.6: MFA enforced for all users
- [ ] CC6.7: System access logging enabled

### System Operations (CC7)
- [ ] CC7.1: Change management via GitOps
- [ ] CC7.2: Infrastructure monitoring active
- [ ] CC7.3: Vulnerability management program

### Risk Management (CC8)
- [ ] CC8.1: Incident response plan documented and tested

### Evidence Collection
- [ ] Configure automated evidence collection
- [ ] Set up audit trail for all administrative actions
- [ ] Document control descriptions and testing procedures

---

## GDPR Requirements

### Data Protection (Art. 25, 32)
- [ ] Encryption at rest and in transit
- [ ] Access controls based on data classification
- [ ] Security monitoring active

### Documentation (Art. 30)
- [ ] Complete data flow mapping
- [ ] Document all processing activities
- [ ] Maintain records of data transfers

### Data Subject Rights (Art. 15-22)
- [ ] Implement DSAR automation
- [ ] Test data export functionality
- [ ] Verify deletion procedures

### Breach Notification (Art. 33)
- [ ] Configure 72-hour notification workflow
- [ ] Set up incident tracking system
- [ ] Train team on breach procedures

### Data Residency
- [ ] Confirm EU data stays in eu-west-1
- [ ] Configure S3 bucket policies for region lock
- [ ] Document any third-party data processors

---

## Penetration Testing

### Pre-Test
- [ ] Submit AWS authorization request
- [ ] Define scope and rules of engagement
- [ ] Set up isolated testing environment
- [ ] Configure alert exclusions for test IPs

### Testing Areas
- [ ] API Gateway endpoints (OWASP API Top 10)
- [ ] EKS cluster (CIS Kubernetes Benchmark)
- [ ] IAM configuration (privilege escalation)
- [ ] Aurora database (SQL injection)

### Post-Test
- [ ] Review findings with development team
- [ ] Prioritize remediation by severity
- [ ] Retest critical/high findings
- [ ] Document lessons learned

---

## Migration Phases

### Pre-Migration Sign-off
- [ ] Security architecture approved
- [ ] IAM design reviewed
- [ ] Network design reviewed
- [ ] Compliance requirements mapped
- [ ] Pen test authorization obtained

### Go-Live Readiness
- [ ] All logging enabled
- [ ] All encryption configured
- [ ] Security monitoring active
- [ ] Incident response tested
- [ ] WAF in BLOCK mode

### Post-Migration (30 Days)
- [ ] Penetration test completed
- [ ] SOC2 evidence collected
- [ ] Security posture baseline established
- [ ] First security review conducted

---

## Approval Sign-off

| Checkpoint | Approver | Date | Status |
|------------|----------|------|--------|
| Security Architecture | Security Lead | | Pending |
| Network Design | Network Team | | Pending |
| IAM Configuration | IAM Admin | | Pending |
| Go-Live Readiness | CISO | | Pending |
| Post-Migration Review | Compliance | | Pending |
