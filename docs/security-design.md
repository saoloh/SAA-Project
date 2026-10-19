# Security Design

## 1. Security objectives

The design uses layered controls to reduce public exposure, limit lateral access, protect database connectivity, and support private administration. These are design intentions; they have not been validated by a live deployment.

## 2. Security group design

Security groups are stateful. Return traffic for an allowed connection is automatically permitted. Prefer references to other security groups for tier-to-tier access rather than broad CIDR rules.

### 2.1 ALB security group

| Direction | Protocol / port | Source or destination | Purpose |
|---|---|---|---|
| Inbound | TCP 443 | CloudFront origin-facing managed prefix list, where supported and configured | Permit HTTPS origin traffic from CloudFront infrastructure |
| Outbound | TCP 80 | Application EC2 security group | Forward requests to application targets |

For initial public testing or where CloudFront is not the only entry point, the inbound source may instead be broader. The production-oriented target in this design is to restrict origin access to CloudFront and validate the origin request using an additional control, such as a protected custom header checked by the ALB WAF. A prefix-list rule alone can permit traffic from other CloudFront distributions; it is not proof that a request came from this particular distribution.

AWS security-group rule quotas and the managed prefix list's rule weight should be checked before implementation.

### 2.2 Application EC2 security group

| Direction | Protocol / port | Source or destination | Purpose |
|---|---|---|---|
| Inbound | TCP 80 | ALB security group | Accept web requests only from the ALB |
| Outbound | TCP 3306 | RDS security group | Connect to MySQL |
| Outbound | TCP 443 | Required external repositories/services, or endpoint security groups as appropriate | Updates and required HTTPS services |

Outbound rules should be narrowed to actual requirements where practical. If an endpoint is used, permit HTTPS to the endpoint's security group as appropriate. The EC2 instances do not need inbound SSH for Session Manager.

### 2.3 RDS security group

| Direction | Protocol / port | Source | Purpose |
|---|---|---|---|
| Inbound | TCP 3306 | Application EC2 security group | Allow application-to-database connections |

Do not allow MySQL from `0.0.0.0/0`. RDS is not publicly accessible. The database security group should not permit direct internet ingress.

### 2.4 Systems Manager endpoint security group

For interface endpoints, permit inbound TCP 443 from the application EC2 security group. Keep endpoint placement, private DNS, IAM permissions, and instance-agent configuration aligned with the services being used.

## 3. AWS WAF and CloudFront origin protection

The design associates a separate AWS WAF web ACL with each of these services:

- **CloudFront web ACL:** uses the CloudFront/global scope and inspects requests at the edge before forwarding them to the origin.
- **ALB web ACL:** uses the ALB's regional scope and inspects requests reaching the load balancer.

The two ACLs can use different or overlapping managed/custom rules according to requirements. Rules should be tested in count mode or a suitable test environment before enforcing potentially disruptive blocks.

### Preventing direct-origin bypass

Because an internet-facing ALB may otherwise be addressed directly, the recommended implementation is to:

1. Restrict the ALB security group's inbound HTTPS rule to the CloudFront origin-facing managed prefix list where supported.
2. Add an origin-verification mechanism, such as a high-entropy custom header configured on the CloudFront origin and validated by an ALB WAF rule.
3. Store and rotate any secret header value securely; do not commit it to a public repository.
4. Test that the intended CloudFront distribution works and direct ALB requests are rejected.

The exact origin restriction should be validated during implementation. Attaching WAF to CloudFront alone does not prevent direct access to a public ALB. WAF also does not replace secure application code, authentication, authorization, or patching.

## 4. TLS and encryption

- Clients use HTTPS to CloudFront.
- CloudFront should use HTTPS to the ALB origin, with a valid certificate and an HTTPS-only origin protocol policy.
- The ALB terminates the origin-side TLS connection and forwards to EC2 over HTTP on TCP port 80 in this design.
- The ALB-to-EC2 hop is not TLS-encrypted. This is a deliberate simplification and should be reconsidered if internal encryption is required by the threat model, policy, or compliance needs.
- RDS storage encryption should be enabled in an implementation, and database connections should use TLS if required by the application security policy.
- Certificates, private keys, database credentials, and origin-verification secrets must not be committed to GitHub.

## 5. Identity and access management

- Attach an IAM instance role to EC2 instead of embedding long-lived AWS access keys.
- Grant only the permissions needed by the application and Systems Manager.
- Use least-privilege human and deployment roles; avoid routine use of root credentials.
- Store database credentials in an appropriate secret-management service, such as AWS Secrets Manager or Systems Manager Parameter Store with suitable protection. Avoid hard-coding credentials in source code or user data.
- Enable MFA for privileged human access where applicable and review IAM permissions periodically.

## 6. Administration and patching

Use Systems Manager Session Manager for administrative access without exposing TCP 22 to the internet. Verify the Systems Manager Agent, instance role, endpoint/network connectivity, and Session Manager permissions.

Apply security updates through a controlled patching process. NAT Gateway access may be needed for external package repositories unless suitable internal mirrors or endpoints are used. Keep application dependencies and the base AMI updated.

## 7. Logging and detection

For an implementation, consider enabling and retaining:
- CloudFront access logs and WAF logs.
- ALB access logs and relevant CloudWatch metrics.
- CloudTrail for AWS API activity.
- VPC Flow Logs where useful for network troubleshooting and detection.
- RDS logs and CloudWatch metrics appropriate to the workload.

Define retention, access permissions, alert thresholds, and response procedures. Logging creates storage and data-processing costs.

## 8. Security limitations

This reference design does not specify a real application's authentication model, authorization rules, data classification, compliance obligations, exact WAF rules, log retention, backup retention, or secret rotation schedule. These must be defined before production use. A network diagram and restrictive security groups are not substitutes for application security testing.
