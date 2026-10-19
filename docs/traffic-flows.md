# Traffic Flows

This document traces the intended network paths. The paths are design descriptions and have not been live-tested.

## 1. Public request to the application

1. The client resolves the application domain using Route 53.
2. DNS directs the client to CloudFront.
3. The client establishes HTTPS to CloudFront.
4. CloudFront applies its AWS WAF web ACL.
5. CloudFront forwards the request to the ALB origin over HTTPS.
6. The ALB web ACL inspects the request.
7. The ALB forwards the request to a healthy EC2 target over HTTP on TCP port 80.
8. The application processes the request and, if necessary, queries RDS.
9. The response returns from EC2 to the ALB, then to CloudFront, and finally to the client.

**Relevant ports:** HTTPS TCP 443 to CloudFront and from CloudFront to ALB; HTTP TCP 80 from ALB to EC2.

## 2. Application to database

1. The application resolves the RDS endpoint.
2. EC2 sends traffic to the database endpoint on TCP port 3306.
3. The VPC local route provides the network path between the subnets, including across AZs.
4. The RDS security group permits the connection only from the application security group.
5. RDS returns the response over the stateful connection.

This traffic does not use NAT Gateway or Internet Gateway. Use the RDS DNS endpoint, not a hard-coded IP address.

## 3. Application instance to the internet

For a package update or an external repository connection:

1. The EC2 instance sends traffic to the destination using its application subnet route table.
2. The `0.0.0.0/0` route sends the traffic to the same-AZ NAT Gateway.
3. The NAT Gateway, located in a public subnet, sends traffic through the Internet Gateway.
4. Return traffic follows the established NAT mapping back to the instance.

The NAT Gateway provides outbound connectivity for private instances; it does not create general unsolicited inbound connectivity to them.

## 4. EC2 to Systems Manager

The Systems Manager Agent uses the instance's IAM role and network connectivity to reach the required Systems Manager service endpoints. With interface VPC endpoints, endpoint traffic uses private VPC connectivity and HTTPS TCP 443. The endpoint security group must permit inbound 443 from the instance security group. The exact endpoint set depends on the Region and services used.

No inbound SSH rule is required for Session Manager.

## 5. Monitoring and notifications

CloudWatch collects supported metrics and evaluates configured alarms. When an alarm's condition is met, it can publish a notification to an SNS topic, which can deliver notifications to subscribed endpoints. Alarm actions and subscriptions must be configured and tested during implementation.

## 6. Origin access considerations

The ALB is internet-facing, but the intended path is CloudFront → ALB. To reduce direct-origin bypass, restrict ALB ingress to the CloudFront origin-facing managed prefix list where supported and validate an origin-verification header with the ALB WAF. The header value must be protected and must not appear in this public documentation. Test both the intended path and direct-origin rejection before treating this control as effective.
