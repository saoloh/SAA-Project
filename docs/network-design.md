# Network Design

## 1. VPC overview

| Setting | Design |
|---|---|
| VPC IPv4 CIDR | `10.0.0.0/16` |
| Addressing | IPv4 |
| Availability Zones | Two AZs |
| Subnet tiers per AZ | Public, private application, private database |
| Application entry | CloudFront → public ALB |
| Application compute | EC2 Auto Scaling group in private application subnets |
| Database | RDS for MySQL in private database subnets |
| Private administration | Systems Manager Session Manager and required connectivity |

The VPC local route allows communication between VPC subnets when the applicable security and network controls permit it. Separate subnets organize network tiers, but subnet separation alone does not block traffic.

## 2. Subnet allocation

| AZ | Subnet tier | CIDR | Purpose |
|---|---|---|---|
| AZ-A | Public | `10.0.1.0/24` | ALB nodes and NAT Gateway A |
| AZ-A | Application private | `10.0.2.0/24` | EC2 application instances |
| AZ-A | Database private | `10.0.3.0/24` | RDS DB subnet group |
| AZ-B | Public | `10.0.4.0/24` | ALB nodes and NAT Gateway B |
| AZ-B | Application private | `10.0.5.0/24` | EC2 application instances |
| AZ-B | Database private | `10.0.6.0/24` | RDS DB subnet group |

AWS reserves some IPv4 addresses in each subnet. The usable address count is therefore lower than the full number implied by the subnet mask. Monitor available addresses as the design grows.

## 3. Routing design

### 3.1 Public route tables

Each public subnet is associated with a route table containing:

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |
| `0.0.0.0/0` | Internet Gateway |

The Internet Gateway is attached to the VPC. Resources that require direct internet reachability also need suitable public addressing and security rules. The ALB is internet-facing; EC2 application instances are not assigned public IP addresses.

### 3.2 Private application route tables

Use a route table per AZ for the application subnets:

| Route table | Destination | Target |
|---|---|---|
| App AZ-A | `10.0.0.0/16` | `local` |
| App AZ-A | `0.0.0.0/0` | NAT Gateway A |
| App AZ-B | `10.0.0.0/16` | `local` |
| App AZ-B | `0.0.0.0/0` | NAT Gateway B |

Each NAT Gateway is deployed in its AZ's public subnet and uses an Elastic IP address. The same-AZ arrangement avoids a normal dependency on the other AZ for outbound internet traffic. If NAT Gateway A fails, the AZ-A route does not automatically switch to NAT Gateway B; any cross-AZ failover routing would need to be designed explicitly.

Where interface VPC endpoints are configured for Systems Manager, traffic to those endpoint services can use the VPC's local routing rather than the NAT Gateway. Other outbound needs, such as software repositories, may still require NAT connectivity.

### 3.3 Database route tables

Database subnets use a route table containing:

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |

There is no default route to a NAT Gateway or Internet Gateway in the base design. RDS is configured as not publicly accessible. Database access is controlled by the RDS security group and the application security group.

## 4. Network traffic flows

### 4.1 Incoming application traffic

1. Route 53 resolves the application domain to CloudFront.
2. The client connects to CloudFront over HTTPS.
3. CloudFront applies its associated AWS WAF web ACL and forwards the request to the ALB origin over HTTPS.
4. The ALB's web ACL applies another layer of request inspection.
5. The ALB forwards requests to healthy EC2 targets over HTTP on TCP port 80.
6. The application reads or writes data in RDS over TCP port 3306 as required.
7. Responses return through the ALB and CloudFront to the client.

### 4.2 Application-to-database traffic

An EC2 instance connects to the RDS endpoint on TCP port 3306. If the instance and RDS are in different AZs but the same VPC, the VPC local route provides the network path. Security groups must allow the connection. No NAT Gateway or Internet Gateway is required for this path.

Use the RDS DNS endpoint rather than a hard-coded database IP address. The endpoint supports connection to the active database endpoint across relevant RDS failover events, although clients must still handle reconnects.

### 4.3 Outbound application traffic

For internet-bound traffic, an application instance sends packets to its AZ's NAT Gateway based on its private route table. The NAT Gateway forwards traffic through the Internet Gateway. The NAT Gateway supports outbound initiation and return traffic; it does not provide general unsolicited inbound access to private instances.

### 4.4 Systems Manager traffic

The EC2 instance uses the Systems Manager Agent and its IAM instance role to establish management connectivity. Required Systems Manager interface endpoints should be deployed in reachable subnets with private DNS and endpoint security-group rules allowing TCP 443 from the instances. The required endpoint set depends on the Region and features used; commonly used endpoints include `ssm` and `ssmmessages`, with `ec2messages` where applicable.

## 5. Design rationale and trade-offs

- **Two AZs:** allows the ALB and application fleet to be distributed across independent AZs. It does not by itself guarantee service continuity.
- **Separate tiers:** makes routing, security-group policies, and IP planning easier to understand and operate.
- **Private application subnets:** avoid direct public IP exposure for EC2 instances.
- **Private DB subnets:** reduce direct exposure and avoid a default internet route; security groups still enforce permitted database connections.
- **NAT Gateway per AZ:** improves AZ-level independence for outbound traffic but increases cost.
- **`/24` subnets:** offer a simple initial allocation; growth and AWS-reserved addresses must be considered.
- **No IPv6:** this design is IPv4-only. IPv6 would require separate routing and security decisions.
