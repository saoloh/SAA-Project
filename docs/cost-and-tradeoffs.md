# Cost and Trade-offs

## 1. Scope

This is a documentation-only reference architecture. No resources have been deployed, and no actual AWS bill or measured cost is claimed. Prices depend on the AWS Region, service configuration, usage, data transfer, storage, retention, and current pricing.

Before implementation, use the AWS Pricing Calculator and review current service pricing for the selected Region.

## 2. Main cost drivers

| Component | Why it may cost money | Design trade-off |
|---|---|---|
| CloudFront | Data transfer, requests, and optional logging/features | Edge delivery and caching versus usage-based charges |
| AWS WAF at CloudFront | Web ACLs, rules, requests, and managed-rule usage may be charged | Edge filtering versus additional security cost |
| AWS WAF at ALB | Regional web ACLs, rules, and requests may be charged | Defense in depth versus duplicated rule and request costs |
| Application Load Balancer | ALB running time and usage-based capacity units | Managed distribution and health checks versus hourly/usage charges |
| EC2 Auto Scaling group | EC2 instance hours, EBS volumes, and related data transfer | Flexible capacity versus compute/storage cost |
| RDS for MySQL Multi-AZ | DB instance capacity, storage, I/O and backups as applicable | Availability/failover versus higher cost than a single-AZ deployment |
| Two NAT Gateways | Gateway hours, data processing, and related transfer charges | AZ-local outbound connectivity versus the cost of two gateways |
| Elastic IP addresses | Applicable public IPv4 address charges | Public IPv4 connectivity for NAT Gateways |
| VPC interface endpoints | Endpoint hours and data processing | Private service connectivity versus endpoint cost |
| Route 53 | Hosted-zone and query charges as applicable | Managed DNS versus service charges |
| CloudWatch and SNS | Metrics, alarms, log ingestion/storage, and notifications according to usage | Observability versus ingestion, retention, and alerting cost |
| Data transfer | Cross-AZ and internet transfer charges may apply | Resilience and distribution versus transfer cost |

This table is qualitative, not a price estimate.

## 3. Availability versus cost

### Two NAT Gateways

One NAT Gateway per AZ reduces the normal cross-AZ dependency for outbound traffic. It costs more than using a single NAT Gateway. A single shared NAT Gateway would lower gateway charges but introduce an AZ dependency and potentially cross-AZ transfer charges.

This design retains two NAT Gateways because AZ-level independence is an explicit goal.

### RDS Multi-AZ

Multi-AZ improves database availability through standby-based failover, but it costs more than a comparable single-AZ deployment. It does not automatically provide read scaling.

This design retains Multi-AZ because availability is a project goal.

### Multiple WAF layers

WAF at CloudFront filters requests at the edge, while WAF at the ALB inspects origin traffic. The two layers add configuration and potentially usage costs. Their rules should be planned to avoid unnecessary duplication and false positives.

### Private service endpoints

Interface endpoints can reduce reliance on NAT for supported AWS service traffic, but endpoint hourly and data-processing charges may apply. Compare the cost of endpoints with NAT processing and connectivity requirements for the actual workload.

## 4. Cost-control measures

- Estimate costs before deployment using the intended Region and realistic usage assumptions.
- Choose EC2 and RDS sizes based on workload evidence; avoid oversized resources.
- Set Auto Scaling minimum and maximum capacity intentionally.
- Review CloudFront caching behavior to reduce unnecessary origin requests where the application permits caching.
- Review log retention and avoid collecting unnecessary high-volume logs.
- Monitor NAT data processing and data-transfer patterns.
- Delete temporary resources after testing and verify that associated resources are also removed.
- Use AWS Budgets or billing alerts where available, while remembering that alerts do not prevent charges.
- Review current free-tier eligibility and limits instead of assuming every service or configuration is free.

## 5. Security and availability trade-offs

Reducing cost by removing a NAT Gateway, switching RDS to Single-AZ, or removing a WAF layer changes the architecture's resilience or security posture. Such changes should be explicit decisions rather than silent substitutions.

## 6. No-deployment statement

The project deliverable documents a proposed architecture and its trade-offs. It does not claim that the design was deployed, load-tested, security-tested, or operated at the costs described. Any future implementation should be validated and costed independently.
