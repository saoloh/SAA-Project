# Availability and Failure Scenarios

## 1. Availability objectives

The design distributes the ALB and application instances across two Availability Zones and uses RDS Multi-AZ for database availability. These choices reduce some single-AZ failure risks, but do not guarantee zero downtime. Actual recovery depends on configuration, health checks, spare capacity, application behavior, and the failure type.

## 2. Availability controls

- **ALB across two AZs:** routes requests to healthy registered targets.
- **EC2 Auto Scaling group across two AZs:** maintains the configured capacity and can replace unhealthy instances when its health checks and policies detect a problem.
- **RDS for MySQL Multi-AZ:** supports standby-based database failover for availability; it is not a read-scaling feature.
- **One NAT Gateway per AZ:** gives each AZ its own normal outbound route.
- **CloudWatch alarms and SNS:** notify operators of selected failures or unhealthy conditions.
- **Route 53 and CloudFront:** provide managed DNS and edge delivery; they do not eliminate origin or application failures.

## 3. Failure scenarios

### 3.1 AZ-A becomes unavailable

Expected behavior:
- ALB operation can continue through the healthy AZ, assuming the remaining ALB capacity and configuration are healthy.
- Application instances in AZ-B can continue serving requests if healthy and sufficiently provisioned.
- The Auto Scaling group may replace or rebalance capacity according to its configuration, but recovery is not instantaneous.
- The AZ-B application subnet continues using NAT Gateway B for outbound internet traffic.
- AZ-A's NAT Gateway becomes unavailable. The AZ-A application route does not automatically change to NAT Gateway B in this design.
- RDS Multi-AZ behavior depends on the database deployment and failure; failover can interrupt existing connections.

Mitigations:
- Keep enough application capacity in the surviving AZ for expected load.
- Configure and test health checks and scaling behavior.
- Ensure critical dependencies are available in the surviving AZ.
- Document operational steps for restoring capacity and investigating the failed AZ.

### 3.2 An EC2 instance becomes unhealthy

Expected behavior:
- ALB health checks stop routing new requests to an unhealthy target after the configured thresholds and intervals.
- If Auto Scaling health checks are configured appropriately, the group can replace an unhealthy instance.
- A target health alarm can notify operators.

The ALB marking a target unhealthy does not, by itself, guarantee that Auto Scaling will replace it. Configure the Auto Scaling group's health-check type and grace period appropriately.

### 3.3 No healthy ALB targets

Expected behavior:
- The application becomes unavailable through the ALB if no healthy targets can serve requests.
- CloudWatch alarms should notify operators.
- Auto Scaling may launch or replace instances if the relevant capacity and health policies detect the problem, but it cannot fix every cause (for example, a broken application release or unavailable shared dependency).

Response:
1. Check target health and ALB metrics.
2. Inspect EC2 status checks, application logs, and recent deployments.
3. Check database and external dependency availability.
4. Roll back a faulty release or restore service according to the incident procedure.

### 3.4 RDS Multi-AZ failover

Expected behavior:
- RDS may fail over to the standby.
- Existing database connections can break; the application should reconnect using the RDS endpoint.
- There may be a period of degraded or interrupted database service.

Application requirements:
- Use bounded retries with backoff for transient connection failures.
- Handle transactions carefully. A client may lose the response after a write has committed, so blind retries can duplicate operations. Use idempotency where appropriate.
- Do not hard-code the database IP address.
- Test reconnection and recovery behavior in a suitable environment before production.

### 3.5 NAT Gateway failure

If NAT Gateway A fails, private instances in AZ-A lose their normal outbound internet path until it is restored or an alternate route is deliberately configured. AZ-B continues to use NAT Gateway B if it remains healthy.

The architecture does not include automatic cross-AZ NAT failover. Cross-AZ routing could be designed as a recovery measure, but it adds routing complexity and potential cross-AZ data-transfer cost.

### 3.6 CloudFront or WAF configuration problem

A misconfigured origin, certificate, cache policy, or WAF rule can block legitimate traffic or make the application unreachable. Use staged changes, monitoring, rule testing, and a documented rollback procedure. WAF rules should be tested before enforcement when false positives are possible.

## 4. Monitoring signals

Examples of useful signals include:
- ALB healthy and unhealthy host counts.
- ALB HTTP 4xx and 5xx metrics and target response time.
- EC2 status checks and application health.
- Auto Scaling desired, in-service, and pending instance counts.
- RDS availability, CPU, connections, free storage, and relevant database metrics.
- NAT Gateway error/drop metrics and data processing indicators.

Alarm thresholds must be selected for the workload and validated; no universal threshold is assumed in this reference design.

## 5. Limits of the design

Multi-AZ resources reduce risk but do not protect against every regional outage, bad deployment, application bug, credential compromise, data corruption, or misconfiguration. Regional disaster recovery, backup retention, restore testing, RTO, and RPO are outside the current scope and should be specified for a production system.
