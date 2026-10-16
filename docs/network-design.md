
# Network Design

## 1. VPC Overview

- VPC CIDR: `10.0.0.0/16`
- Availability Zones: 2
- Subnet strategy: Public, private application, and private database subnets in each AZ.
- IP addressing: IPv4

## 2. Subnet Allocation

| Availability Zone | Subnet Type | CIDR |
|---|---|---|
| AZ-A | Public | `10.0.1.0/24` |
| AZ-A | Application Private | `10.0.2.0/24` |
| AZ-A | Database Private | `10.0.3.0/24` |
| AZ-B | Public | `10.0.4.0/24` |
| AZ-B | Application Private | `10.0.5.0/24` |
| AZ-B | Database Private | `10.0.6.0/24` |

## 3. Routing Design

### Public Route Tables

- Local VPC route: `10.0.0.0/16`
- Default route: `0.0.0.0/0` → Internet Gateway

### Application Private Route Tables

- Local VPC route: `10.0.0.0/16`
- Default route: `0.0.0.0/0` → NAT Gateway in the same AZ

### Database Private Route Tables

- Local VPC route: `10.0.0.0/16`
- No default internet route in the basic design

## 4. Network Traffic

Explain the paths for:
- Incoming web requests
- Application-to-database connections
- Outbound internet access from EC2
- Private administrative access through Systems Manager

## 5. Design Rationale

Explain why the design uses:
- Two Availability Zones
- Separate application and database subnets
- A NAT Gateway per AZ
- Private database subnets
