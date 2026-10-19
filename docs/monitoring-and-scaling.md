# Monitoring and Scaling

## 1. Monitoring approach

Amazon CloudWatch is the central metrics and alarm service in this design. Amazon SNS delivers notifications from configured alarm actions to subscribed recipients. Exact metrics, thresholds, destinations, and retention periods must be selected and tested during implementation.

## 2. Application load balancing and health

Monitor:
- Healthy and unhealthy ALB target counts.
- Request count, target response time, and HTTP 5xx metrics.
- EC2 status checks and application-level health.
- Auto Scaling group capacity and instance lifecycle state.

Configure ALB health checks to test whether an application target can serve requests. Choose a health-check path and thresholds that reflect meaningful application health without making transient conditions cause unnecessary target removal.

An ALB target becoming unhealthy does not automatically guarantee an EC2 instance will be replaced. Configure Auto Scaling health-check integration and grace periods appropriately.

## 3. EC2 Auto Scaling

The intended scaling policy is **target tracking based on ALB requests per target**. The Auto Scaling group should span the two application subnets and have explicitly configured minimum, desired, and maximum capacity.

Considerations:
- Set the target value using expected application behavior and testing, not an arbitrary universal number.
- Ensure the maximum capacity and instance types can support the intended load.
- Allow for instance launch and application warm-up time.
- Ensure the application can run on multiple instances without relying on instance-local persistent data.
- Test scale-out and scale-in behavior, including graceful handling of in-flight requests.

Scaling is not a substitute for fixing slow database queries, failing dependencies, or defective application releases.

## 4. RDS monitoring

Monitor database availability, CPU, connections, storage, latency, and workload-specific metrics.

Example responses:
- **Low free storage:** notify and investigate storage growth; plan a suitable RDS storage modification or capacity change. Do not scale EC2 in response to a database storage issue.
- **High database CPU or connections:** investigate query performance, connection pooling, indexes, and database capacity before changing application capacity.
- **Database failover or connectivity loss:** verify RDS events and application reconnection behavior.

RDS Multi-AZ is for availability and failover. It does not by itself distribute read traffic across replicas.

## 5. NAT and network monitoring

Monitor NAT Gateway metrics relevant to errors, dropped traffic, and traffic volume. Investigate failures in outbound repository access or Systems Manager connectivity. If interface endpoints are used, verify endpoint health, DNS resolution, routing, and endpoint security-group rules.

## 6. Example alarm-response matrix

| Signal | Notification / response | What not to assume |
|---|---|---|
| Zero healthy ALB targets | Urgent notification; inspect target health, app release, and dependencies | Do not assume Auto Scaling will always repair the root cause |
| Rising ALB 5xx | Notify and investigate target/application errors; scale only if capacity data supports it | 5xx does not automatically mean more EC2 instances are needed |
| High request count per target | Target-tracking policy can scale the application fleet within configured bounds | Scaling is constrained by quotas, launch time, and max capacity |
| Low RDS free storage | Notify and plan a database storage action | EC2 scaling will not fix database storage pressure |
| RDS connectivity/failover event | Notify; check RDS events and application reconnect behavior | Multi-AZ does not guarantee uninterrupted existing connections |
| EC2 status-check failure | Notify; inspect the instance and Auto Scaling health settings | An ALB health check alone does not guarantee instance replacement |
| NAT Gateway errors or loss of egress | Notify and investigate the affected AZ's outbound path | The other AZ's NAT Gateway is not automatically used by this AZ |

## 7. Notifications and operations

CloudWatch alarm actions can publish to an SNS topic. Subscribe the intended recipients and confirm delivery during implementation. Establish a clear response owner, severity, and runbook for each critical alarm to reduce alert fatigue.

## 8. Logging

Consider CloudFront, WAF, and ALB access logs; CloudTrail for API activity; VPC Flow Logs for network troubleshooting; application logs; and RDS logs. Define retention, access control, and sensitive-data handling before enabling production logging. Logging and data ingestion may incur charges.
