# CloudFormation Nested Stack Template: VPC Subnets with Multi-AZ NAT

<!-- Row 1: Status - Most Important -->
[![Release](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)&nbsp;[![GitHub Action](https://img.shields.io/badge/GitHub-Action-blue?logo=github)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/github/languages/top/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazonaws&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ha-multi-az-vpc-subnets/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/ecdcf4fd75b99f1f6dd3162737bec745/raw/cfn-nested-aws-ha-multi-az-vpc-subnets.json?)](https://gist.github.com/bsubhamay/ecdcf4fd75b99f1f6dd3162737bec745)

---

## Overview

This repository contains a **CloudFormation nested stack template** for deploying highly available private subnets across multiple AWS Availability Zones. The template provides reusable infrastructure for creating private subnet tiers with optional multi-AZ NAT Gateways, independent route tables per Availability Zone, and Network ACLs for security isolation.

**Key characteristics:**

- ✅ **Multi-AZ Support** — Create 1-4 private subnets across different Availability Zones
- ✅ **One Route Table Per Subnet** — Independent AZ-level routing control
- ✅ **Optional Multi-AZ NAT Gateways** — High-availability internet egress with Elastic IPs
- ✅ **Network ACLs** — Optional stateless firewall rules for subnet isolation
- ✅ **Parameterized Configuration** — Flexible deployments for dev, staging, and production
- ✅ **Cross-Stack References** — Export outputs for parent stack consumption

## Architecture

### Private Subnet Tier

The template creates a complete private subnet tier with:

1. **Private Subnets (1-4)** — Isolated subnets without direct internet access
2. **Route Tables (One per Subnet)** — Individual routing policies per Availability Zone
3. **NAT Gateways (Optional)** — Single or multi-AZ configuration for internet egress
4. **Network ACLs (Optional)** — Stateless security rules for inbound/outbound traffic

### Why One Route Table Per Subnet?

This design enables:

- **Independent AZ routing** — Different NAT Gateways in different AZs
- **Granular traffic control** — Fine-grained routing policies per availability zone
- **Compliance requirements** — Predictable traffic paths for audit trails

### Multi-AZ NAT Patterns

**Single NAT (Cost-Optimized):**

```yaml
EnableNATGateway: "true"
EnableHighAvailabilityNAT: "false"  # One NAT in first AZ
```

Lower cost but single point of failure. Suitable for development/test environments.

**Multi-AZ NAT (High Availability):**

```yaml
EnableNATGateway: "true"
EnableHighAvailabilityNAT: "true"   # One NAT per AZ
```

Redundancy with no cross-AZ data transfer charges. Recommended for production.

## Template Files

### CloudFormation Templates

- **`templates/vpc-subnets.yaml`** — Main nested template for creating private subnets with optional NAT Gateways and NACLs

### Parameter Files

- **`parameters/parameters.json`** — Template parameter values for deployments

## Parameters

| Parameter | Type | Required | Description |
| ----------- | ------ | --------- | ------------- |
| `EnvironmentName` | String | Yes | Environment name prefix (e.g., `prod`, `staging`, `dev`) |
| `VPCId` | AWS::EC2::VPC::Id | Yes | Existing VPC to attach subnets to |
| `SubnetCount` | Number | Yes | Number of private subnets (1-4, one per AZ) |
| `SubnetCidrBlocks` | CommaDelimitedList | Yes | CIDR blocks for subnets (e.g., `10.0.1.0/24,10.0.2.0/24`) |
| `AvailabilityZones` | CommaDelimitedList | Yes | AZs for subnets (e.g., `us-east-1a,us-east-1b,us-east-1c`) |
| `EnableNATGateway` | String | Yes | Enable NAT Gateway for internet egress (`true`/`false`) |
| `EnableHighAvailabilityNAT` | String | Yes | Create one NAT Gateway per AZ (`true`/`false`); requires `EnableNATGateway=true` |
| `PublicSubnetIds` | CommaDelimitedList | Yes | Public subnet IDs where NAT Gateways will be placed (must match subnet count) |
| `EnableNACLs` | String | No | Enable Network ACLs (default: `true`) |

## Outputs

- **`SubnetIds`** — Private subnet IDs (comma-separated)
- **`RouteTableIds`** — Route table IDs (comma-separated)
- **`NATGatewayIds`** — NAT Gateway IDs (if enabled)
- **`NATGatewayIps`** — Elastic IPs for NAT Gateways (if enabled)
- **`NetworkAclId`** — Network ACL ID (if enabled)
- **`SubnetCount`** — Number of subnets created

## Usage

### 1. Upload Template to S3

```bash
aws s3 cp templates/vpc-subnets.yaml s3://your-cfn-bucket/templates/vpc-subnets.yaml
```

### 2. Reference from Parent Stack

In your parent CloudFormation template:

```yaml
PrivateSubnetsNestedStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/vpc-subnets.yaml
    Parameters:
      EnvironmentName: !Ref Environment
      VPCId: !Ref VpcId
      SubnetCount: "2"
      SubnetCidrBlocks: !Join [",", ["10.0.1.0/24", "10.0.2.0/24"]]
      AvailabilityZones: !Join [",", ["us-east-1a", "us-east-1b"]]
      EnableNATGateway: "true"
      EnableHighAvailabilityNAT: "true"
      PublicSubnetIds: !Join [",", [!Ref PublicSubnet1, !Ref PublicSubnet2]]
      EnableNACLs: "true"
    Tags:
      - Key: Environment
        Value: !Ref Environment

Outputs:
  PrivateSubnetIds:
    Value: !GetAtt PrivateSubnetsNestedStack.Outputs.SubnetIds
    Export:
      Name: !Sub "${AWS::StackName}-PrivateSubnetIds"
  NATGatewayIps:
    Value: !GetAtt PrivateSubnetsNestedStack.Outputs.NATGatewayIps
    Export:
      Name: !Sub "${AWS::StackName}-NATGatewayIps"
```

### 3. Deploy Using AWS CLI

#### Option A: Deploy 2-AZ Private Subnet Stack

```bash
aws cloudformation create-stack \
  --stack-name prod-private-subnets \
  --template-body file://templates/vpc-subnets.yaml \
  --parameters file://parameters/prod-2az.json \
  --region us-east-1
```

#### Option B: Deploy with CLI Parameters

```bash
aws cloudformation create-stack \
  --stack-name prod-private-subnets-2az \
  --template-body file://templates/vpc-subnets.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=prod \
    ParameterKey=VPCId,ParameterValue=vpc-0123456789abcdef0 \
    ParameterKey=SubnetCount,ParameterValue=2 \
    ParameterKey=SubnetCidrBlocks,ParameterValue="10.0.1.0/24,10.0.2.0/24" \
    ParameterKey=AvailabilityZones,ParameterValue="us-east-1a,us-east-1b" \
    ParameterKey=EnableNATGateway,ParameterValue=true \
    ParameterKey=EnableHighAvailabilityNAT,ParameterValue=true \
    ParameterKey=PublicSubnetIds,ParameterValue="subnet-public-1a,subnet-public-1b" \
    ParameterKey=EnableNACLs,ParameterValue=true
```

#### Option C: Monitor Stack Creation

```bash
# Wait for stack to complete
aws cloudformation wait stack-create-complete \
  --stack-name prod-private-subnets-2az \
  --region us-east-1

# View outputs
aws cloudformation describe-stacks \
  --stack-name prod-private-subnets-2az \
  --region us-east-1 \
  --query 'Stacks[0].Outputs'
```

### 4. Update Existing Stack

Enable NAT Gateways on an existing deployment:

```bash
aws cloudformation update-stack \
  --stack-name prod-private-subnets \
  --template-body file://templates/vpc-subnets.yaml \
  --parameters file://parameters/prod-with-nat.json \
  --region us-east-1
```

## Use Cases & Parameter Combinations

### Use Case 1: High-Availability Production VPC (4-AZ)

**Scenario:** Multi-region capable production environment with maximum redundancy and isolation

**Key Parameters:**

- `PublicSubnetCount: 4` — Public tier across all AZs for load balancers
- `PrivateSubnetCount: 4` — Private tier across all AZs for application servers
- `EnableNATGateway: true` + `EnableHighAvailabilityNAT: true` — One NAT per AZ, no cross-AZ data transfer charges
- `EnableNACLs: true` — Stateless firewall rules for defense-in-depth
- `EnableDnsHostnames: true` — Required for service discovery
- `MapPublicIpOnLaunch: false` — Explicit control over public IPs (use Elastic IPs)

**Benefits:**

- ✅ Zero single points of failure (all AZs independent)
- ✅ Optimal data transfer costs (NAT traffic stays in AZ)
- ✅ Compliance-friendly (predictable routing paths)
- ✅ Supports auto-scaling across AZs

---

### Use Case 2: Cost-Optimized Development VPC (2-AZ with Single NAT)

**Scenario:** Development environment with redundancy but minimal costs

**Key Parameters:**

- `PublicSubnetCount: 2` — Public tier in 2 AZs for HA
- `PrivateSubnetCount: 2` — Private tier in 2 AZs
- `EnableNATGateway: true` + `EnableHighAvailabilityNAT: false` — Single NAT Gateway (first AZ only)
- `EnableNACLs: true` — Basic security isolation
- `EnableDnsHostnames: true` — Standard setup

**Benefits:**

- ✅ Lower costs (only 1 NAT Gateway + 1 Elastic IP)
- ✅ Adequate HA for non-critical workloads
- ✅ Cross-AZ data transfer charges apply but acceptable for dev
- ✅ Quick provisioning and tear-down

---

### Use Case 3: Private-Only VPC (No Internet Egress)

**Scenario:** Isolated environment for databases, caches, or airgapped deployments

**Key Parameters:**

- `PublicSubnetCount: 0` — No public tier
- `PrivateSubnetCount: 3` — Private tier in 3 AZs only
- `EnableInternetGateway: false` — No Internet Gateway
- `EnableNATGateway: false` — No internet egress
- `EnableNACLs: true` — Network isolation
- `EnableDnsHostnames: true` — Internal service discovery

**Benefits:**

- ✅ Ultra-secure (zero internet exposure)
- ✅ Lowest cost (no NAT/IGW charges)
- ✅ Ideal for backend databases and data stores
- ✅ VPC endpoints used for AWS service access

---

### Use Case 4: Web Application Tier (Public+Private with HTTP/HTTPS)

**Scenario:** Classic 3-tier architecture with load balancers and app servers

**Key Parameters:**

- `PublicSubnetCount: 2` — Public tier for ALB/NLB load balancers
- `PrivateSubnetCount: 2` — Private tier for application servers
- `EnableNATGateway: true` + `EnableHighAvailabilityNAT: true` — Multi-AZ NAT for app egress
- `AllowHTTPFromInternet: true` — Port 80 allowed
- `AllowHTTPSFromInternet: true` — Port 443 allowed
- `AllowSSHFromPrivate: true` — SSH from private to public bastion hosts
- `MapPublicIpOnLaunch: false` — Use Elastic IPs on NAT Gateways instead

**Benefits:**

- ✅ Proper network segmentation (public/private)
- ✅ Load balancers in public subnets, apps in private
- ✅ HA across AZs with independent NATs
- ✅ Firewall rules allow standard web protocols

---

### Use Case 5: Single-AZ Development Cluster (Minimal)

**Scenario:** Rapid prototyping or single-AZ development environment

**Key Parameters:**

- `PublicSubnetCount: 1` — Single public subnet
- `PrivateSubnetCount: 1` — Single private subnet
- `AvailabilityZone: us-east-1a` (any single AZ)
- `EnableNATGateway: true` + `EnableHighAvailabilityNAT: false` — Single NAT (if needed)
- `EnableNACLs: false` — Disabled for simpler troubleshooting
- `MapPublicIpOnLaunch: true` — Auto-assign public IPs

**Benefits:**

- ✅ Minimal resource consumption
- ✅ Fastest deployment
- ✅ Suitable for POCs and experiments
- ✅ Easy to destroy and recreate

---

### Use Case 6: Microservices Platform (3-AZ with Service Discovery)

**Scenario:** Kubernetes or container orchestration platform with internal DNS

**Key Parameters:**

- `PublicSubnetCount: 3` — Public tier for ingress controllers
- `PrivateSubnetCount: 3` — Private tier for container workloads
- `EnableDnsHostnames: true` — Required for service discovery (e.g., EKS, ECS)
- `EnableNATGateway: true` + `EnableHighAvailabilityNAT: true` — Multi-AZ NAT for outbound
- `AllowHTTPFromInternet: true` — Ingress on port 80
- `AllowHTTPSFromInternet: true` — Ingress on port 443
- `EnableNACLs: true` — Network policy enforcement

**Benefits:**

- ✅ DNS resolution enables service discovery
- ✅ Multi-AZ deployment for container orchestrators
- ✅ Public ingress controllers in public subnets
- ✅ Pod/container workloads in private subnets

---

## Example Parameter Files

### 4-AZ Production with Multi-AZ NAT

```json
[
  {
    "ParameterKey": "ApplicationPrefix",
    "ParameterValue": "myapp"
  },
  {
    "ParameterKey": "EnvironmentName",
    "ParameterValue": "prod"
  },
  {
    "ParameterKey": "VPCCidrBlock",
    "ParameterValue": "10.0.0.0/16"
  },
  {
    "ParameterKey": "EnableInternetGateway",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableDnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "PublicSubnetCount",
    "ParameterValue": "4"
  },
  {
    "ParameterKey": "PublicSubnetCidrBlocks",
    "ParameterValue": "10.0.1.0/24,10.0.2.0/24,10.0.3.0/24,10.0.4.0/24"
  },
  {
    "ParameterKey": "PublicSubnetAZ1",
    "ParameterValue": "us-east-1a"
  },
  {
    "ParameterKey": "PublicSubnetAZ2",
    "ParameterValue": "us-east-1b"
  },
  {
    "ParameterKey": "PublicSubnetAZ3",
    "ParameterValue": "us-east-1c"
  },
  {
    "ParameterKey": "PublicSubnetAZ4",
    "ParameterValue": "us-east-1d"
  },
  {
    "ParameterKey": "MapPublicIpOnLaunch",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "PrivateSubnetCount",
    "ParameterValue": "4"
  },
  {
    "ParameterKey": "PrivateSubnetCidrBlocks",
    "ParameterValue": "10.0.11.0/24,10.0.12.0/24,10.0.13.0/24,10.0.14.0/24"
  },
  {
    "ParameterKey": "PrivateSubnetAZ1",
    "ParameterValue": "us-east-1a"
  },
  {
    "ParameterKey": "PrivateSubnetAZ2",
    "ParameterValue": "us-east-1b"
  },
  {
    "ParameterKey": "PrivateSubnetAZ3",
    "ParameterValue": "us-east-1c"
  },
  {
    "ParameterKey": "PrivateSubnetAZ4",
    "ParameterValue": "us-east-1d"
  },
  {
    "ParameterKey": "EnableNATGateway",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableHighAvailabilityNAT",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableNACLs",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "AllowHTTPFromInternet",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "AllowHTTPSFromInternet",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "AllowSSHFromPrivate",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "AllowPingFromPrivate",
    "ParameterValue": "true"
  }
]
```

### 2-AZ Development with Single NAT (Cost-Optimized)

```json
[
  {
    "ParameterKey": "ApplicationPrefix",
    "ParameterValue": "devapp"
  },
  {
    "ParameterKey": "EnvironmentName",
    "ParameterValue": "dev"
  },
  {
    "ParameterKey": "VPCCidrBlock",
    "ParameterValue": "10.1.0.0/16"
  },
  {
    "ParameterKey": "EnableInternetGateway",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableDnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "PublicSubnetCount",
    "ParameterValue": "2"
  },
  {
    "ParameterKey": "PublicSubnetCidrBlocks",
    "ParameterValue": "10.1.1.0/24,10.1.2.0/24"
  },
  {
    "ParameterKey": "PublicSubnetAZ1",
    "ParameterValue": "us-east-1a"
  },
  {
    "ParameterKey": "PublicSubnetAZ2",
    "ParameterValue": "us-east-1b"
  },
  {
    "ParameterKey": "MapPublicIpOnLaunch",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "PrivateSubnetCount",
    "ParameterValue": "2"
  },
  {
    "ParameterKey": "PrivateSubnetCidrBlocks",
    "ParameterValue": "10.1.11.0/24,10.1.12.0/24"
  },
  {
    "ParameterKey": "PrivateSubnetAZ1",
    "ParameterValue": "us-east-1a"
  },
  {
    "ParameterKey": "PrivateSubnetAZ2",
    "ParameterValue": "us-east-1b"
  },
  {
    "ParameterKey": "EnableNATGateway",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableHighAvailabilityNAT",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "EnableNACLs",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "AllowHTTPFromInternet",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "AllowHTTPSFromInternet",
    "ParameterValue": "true"
  }
]
```

### Private-Only VPC (Databases/Cache Layer)

```json
[
  {
    "ParameterKey": "ApplicationPrefix",
    "ParameterValue": "data"
  },
  {
    "ParameterKey": "EnvironmentName",
    "ParameterValue": "prod"
  },
  {
    "ParameterKey": "VPCCidrBlock",
    "ParameterValue": "10.2.0.0/16"
  },
  {
    "ParameterKey": "EnableInternetGateway",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "EnableDnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "PublicSubnetCount",
    "ParameterValue": "0"
  },
  {
    "ParameterKey": "PublicSubnetCidrBlocks",
    "ParameterValue": ""
  },
  {
    "ParameterKey": "PrivateSubnetCount",
    "ParameterValue": "3"
  },
  {
    "ParameterKey": "PrivateSubnetCidrBlocks",
    "ParameterValue": "10.2.1.0/24,10.2.2.0/24,10.2.3.0/24"
  },
  {
    "ParameterKey": "PrivateSubnetAZ1",
    "ParameterValue": "us-east-1a"
  },
  {
    "ParameterKey": "PrivateSubnetAZ2",
    "ParameterValue": "us-east-1b"
  },
  {
    "ParameterKey": "PrivateSubnetAZ3",
    "ParameterValue": "us-east-1c"
  },
  {
    "ParameterKey": "EnableNATGateway",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "EnableNACLs",
    "ParameterValue": "true"
  }
]
```

### Single-AZ Development Prototype

```json
[
  {
    "ParameterKey": "ApplicationPrefix",
    "ParameterValue": "poc"
  },
  {
    "ParameterKey": "EnvironmentName",
    "ParameterValue": "poc"
  },
  {
    "ParameterKey": "VPCCidrBlock",
    "ParameterValue": "10.3.0.0/16"
  },
  {
    "ParameterKey": "EnableInternetGateway",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableDnsHostnames",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "PublicSubnetCount",
    "ParameterValue": "1"
  },
  {
    "ParameterKey": "PublicSubnetCidrBlocks",
    "ParameterValue": "10.3.1.0/24"
  },
  {
    "ParameterKey": "PublicSubnetAZ1",
    "ParameterValue": "us-east-1a"
  },
  {
    "ParameterKey": "MapPublicIpOnLaunch",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "PrivateSubnetCount",
    "ParameterValue": "1"
  },
  {
    "ParameterKey": "PrivateSubnetCidrBlocks",
    "ParameterValue": "10.3.11.0/24"
  },
  {
    "ParameterKey": "PrivateSubnetAZ1",
    "ParameterValue": "us-east-1a"
  },
  {
    "ParameterKey": "EnableNATGateway",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "EnableNACLs",
    "ParameterValue": "false"
  }
]
```

## Best Practices

- ✅ **Multi-AZ Redundancy** — Use multi-AZ NAT for production workloads
- ✅ **Independent Route Tables** — One route table per subnet enables AZ-level control
- ✅ **Network ACLs** — Enable NACLs for stateless security policies
- ✅ **Proper CIDR Planning** — Ensure subnet CIDR blocks don't overlap with other VPC tiers
- ✅ **Elastic IPs** — NAT Gateways use Elastic IPs for consistent outbound addresses
- ✅ **Cross-Stack References** — Export subnet IDs and route table IDs for dependent stacks

## Cleanup

```bash
aws cloudformation delete-stack \
  --stack-name prod-private-subnets \
  --region us-east-1
```

## License

MIT
