# Parameter Files Guide

This document explains each parameter file and the VPC architectures they represent.

## Overview of Parameter Files

All parameter files use **non-overlapping CIDR ranges** to allow simultaneous deployment of multiple stacks for testing:

| File | VPC CIDR | Use Case | Subnets |
|------|----------|----------|---------|
| `vpc-single-private-subnet.json` | `10.0.0.0/16` | Dev/test with private only | 1 private |
| `vpc-single-public-subnet.json` | `10.1.0.0/16` | Public web tier only | 1 public |
| `vpc-public-private-subnets.json` | `10.2.0.0/16` | Standard 2-tier app | 1 pub + 1 priv |
| `vpc-two-public-private-subnets.json` | `10.3.0.0/16` | Multi-AZ 2-tier | 2 pub + 2 priv |
| `vpc-two-public-four-private-subnets.json` | `10.4.0.0/16` | HA multi-AZ production | 2 pub + 4 priv |

## Detailed Architecture Breakdown

### 1. `vpc-single-private-subnet.json`

**Architecture:**
```
VPC: 10.0.0.0/16
└── Private Subnet 1: 10.0.1.0/24 (us-east-1a)
    └── No internet access
    └── No NAT Gateway
```

**Use Case:** 
- Development/testing environment
- Isolated workloads that don't need internet
- Minimal cost setup

**Key Parameters:**
```json
{
  "EnvironmentName": "single-private",
  "VPCCidrBlock": "10.0.0.0/16",
  "EnableInternetGateway": "false",
  "PrivateSubnetCount": "1",
  "PrivateSubnetCidrBlocks": "10.0.1.0/24",
  "PrivateAvailabilityZones": "us-east-1a",
  "EnableNATGateway": "false"
}
```

**Deployment:**
```bash
aws cloudformation create-stack \
  --stack-name dev-private-vpc \
  --template-body file://templates/parent-stack.yaml \
  --parameters file://parameters/vpc-single-private-subnet.json \
  --region us-east-1
```

---

### 2. `vpc-single-public-subnet.json`

**Architecture:**
```
VPC: 10.1.0.0/16
├── Internet Gateway
└── Public Subnet 1: 10.1.1.0/24 (us-east-1a)
    ├── Auto-assign public IPs
    └── Route to IGW: 0.0.0.0/0 → IGW
```

**Use Case:**
- Public-facing applications only
- Simple single-server deployments
- Bastion hosts

**Key Parameters:**
```json
{
  "EnvironmentName": "single-public",
  "VPCCidrBlock": "10.1.0.0/16",
  "EnableInternetGateway": "true",
  "PublicSubnetCount": "1",
  "PublicSubnetCidrBlocks": "10.1.1.0/24",
  "PublicAvailabilityZones": "us-east-1a",
  "AllowHTTPFromInternet": "true",
  "AllowHTTPSFromInternet": "true"
}
```

**Security Groups:**
- HTTP (80) and HTTPS (443) from internet
- All outbound traffic allowed

---

### 3. `vpc-public-private-subnets.json`

**Architecture:**
```
VPC: 10.2.0.0/16
├── Internet Gateway
├── Public Subnet 1: 10.2.1.0/24 (us-east-1a)
│   ├── NAT Gateway 1 (single)
│   └── Route: 0.0.0.0/0 → IGW
└── Private Subnet 1: 10.2.2.0/24 (us-east-1a)
    └── Route: 0.0.0.0/0 → NAT Gateway 1
```

**Use Case:**
- Standard 2-tier web application (web tier + database tier)
- Single AZ deployment
- Good for dev/staging environments

**Key Parameters:**
```json
{
  "EnvironmentName": "pub-priv-1az",
  "VPCCidrBlock": "10.2.0.0/16",
  "EnableInternetGateway": "true",
  "PublicSubnetCount": "1",
  "PublicSubnetCidrBlocks": "10.2.1.0/24",
  "PrivateSubnetCount": "1",
  "PrivateSubnetCidrBlocks": "10.2.2.0/24",
  "EnableNATGateway": "true",
  "EnableHighAvailabilityNAT": "false"
}
```

**Security Groups:**
- Public SG: HTTP/HTTPS from internet, SSH/PING from private
- Private SG: All traffic from public, self-to-self

**Traffic Flow:**
```
Internet → IGW → Web Tier (public) ↔ App Tier (private) → NAT → Internet
```

---

### 4. `vpc-two-public-private-subnets.json`

**Architecture:**
```
VPC: 10.3.0.0/16
├── Internet Gateway
├── Public Subnets (Multi-AZ)
│   ├── Public Subnet 1: 10.3.1.0/24 (us-east-1a)
│   │   ├── NAT Gateway 1 (single, cost-optimized)
│   │   └── Route: 0.0.0.0/0 → IGW
│   └── Public Subnet 2: 10.3.2.0/24 (us-east-1b)
│       └── Route: 0.0.0.0/0 → IGW
└── Private Subnets (Multi-AZ)
    ├── Private Subnet 1: 10.3.3.0/24 (us-east-1a)
    │   └── Route: 0.0.0.0/0 → NAT Gateway 1
    └── Private Subnet 2: 10.3.4.0/24 (us-east-1b)
        └── Route: 0.0.0.0/0 → NAT Gateway 1
```

**Use Case:**
- Multi-AZ deployment for HA
- Cost-optimized (single NAT, but not recommended for production)
- Production-like staging environment

**Key Parameters:**
```json
{
  "EnvironmentName": "pub-priv-2az",
  "VPCCidrBlock": "10.3.0.0/16",
  "EnableInternetGateway": "true",
  "PublicSubnetCount": "2",
  "PublicSubnetCidrBlocks": "10.3.1.0/24,10.3.2.0/24",
  "PublicAvailabilityZones": "us-east-1a,us-east-1b",
  "PrivateSubnetCount": "2",
  "PrivateSubnetCidrBlocks": "10.3.3.0/24,10.3.4.0/24",
  "PrivateAvailabilityZones": "us-east-1a,us-east-1b",
  "EnableNATGateway": "true",
  "EnableHighAvailabilityNAT": "false"
}
```

**Trade-offs:**
- ✅ HA within a region (2 AZs)
- ✅ Lower cost ($0.32/hour for one NAT)
- ❌ Cross-AZ data transfer for private tier egress ($0.01/GB)
- ❌ Single NAT is a bottleneck if it fails

**Traffic Flow:**
```
Internet
  ↓
IGW
  ├→ Public Subnet 1a (us-east-1a) ↔ Private Subnet 1a (us-east-1a)
  └→ Public Subnet 1b (us-east-1b) ↔ Private Subnet 1b (us-east-1b)
                                         ↓
                                  NAT Gateway 1 → Internet
```

---

### 5. `vpc-two-public-four-private-subnets.json`

**Architecture:**
```
VPC: 10.4.0.0/16
├── Internet Gateway
├── Public Subnets (2 AZs)
│   ├── Public Subnet 1a: 10.4.1.0/24 (us-east-1a)
│   │   ├── NAT Gateway 1
│   │   └── Route: 0.0.0.0/0 → IGW
│   └── Public Subnet 2b: 10.4.2.0/24 (us-east-1b)
│       ├── NAT Gateway 2
│       └── Route: 0.0.0.0/0 → IGW
└── Private Subnets (4 AZs)
    ├── Private Subnet 1a: 10.4.3.0/24 (us-east-1a)
    │   ├── Route Table 1a → NAT Gateway 1
    ├── Private Subnet 2b: 10.4.4.0/24 (us-east-1b)
    │   ├── Route Table 2b → NAT Gateway 2
    ├── Private Subnet 3c: 10.4.5.0/24 (us-east-1c)
    │   ├── Route Table 3c → NAT Gateway 2 (or 1, depending on config)
    └── Private Subnet 4d: 10.4.6.0/24 (us-east-1d)
        └── Route Table 4d → NAT Gateway 2 (or 1)
```

**Use Case:**
- **Production-grade multi-AZ HA deployment**
- One NAT Gateway per AZ (when EnableHighAvailabilityNAT=true)
- Highly available and cost-optimized

**Key Parameters:**
```json
{
  "EnvironmentName": "pub-priv-4az",
  "VPCCidrBlock": "10.4.0.0/16",
  "EnableInternetGateway": "true",
  "PublicSubnetCount": "2",
  "PublicSubnetCidrBlocks": "10.4.1.0/24,10.4.2.0/24",
  "PublicAvailabilityZones": "us-east-1a,us-east-1b",
  "PrivateSubnetCount": "4",
  "PrivateSubnetCidrBlocks": "10.4.3.0/24,10.4.4.0/24,10.4.5.0/24,10.4.6.0/24",
  "PrivateAvailabilityZones": "us-east-1a,us-east-1b,us-east-1c,us-east-1d",
  "EnableNATGateway": "true",
  "EnableHighAvailabilityNAT": "true"
}
```

**Advantages:**
- ✅ HA NAT per AZ (no single point of failure)
- ✅ No cross-AZ data transfer charges
- ✅ Scales to 4 AZs for database layer
- ✅ Each AZ is independent

**Cost Analysis:**
- NAT Gateway HA: $0.32/hour × 2 = $0.64/hour ($470/month)
- Better for production with high throughput

---

## Using Parameter Files

### Deploy a Single Stack

```bash
# Deploy the 2-public + 4-private architecture
aws cloudformation create-stack \
  --stack-name prod-vpc-stack \
  --template-body file://templates/parent-stack.yaml \
  --parameters file://parameters/vpc-two-public-four-private-subnets.json \
  --region us-east-1
```

### Deploy Multiple Stacks Simultaneously

Since CIDR ranges don't overlap, deploy all five for comparison:

```bash
for param_file in parameters/vpc-*.json; do
  stack_name=$(basename "$param_file" .json)
  aws cloudformation create-stack \
    --stack-name "$stack_name" \
    --template-body file://templates/parent-stack.yaml \
    --parameters file://"$param_file" \
    --region us-east-1
done
```

### Update a Stack

```bash
aws cloudformation update-stack \
  --stack-name pub-priv-2az \
  --template-body file://templates/parent-stack.yaml \
  --parameters file://parameters/vpc-two-public-private-subnets.json \
  --region us-east-1
```

### Monitor Stack Events

```bash
aws cloudformation describe-stack-events \
  --stack-name pub-priv-4az \
  --region us-east-1 \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

---

## Security Group Rules by Architecture

### Single Public Subnet
- **HTTP (80):** 0.0.0.0/0
- **HTTPS (443):** 0.0.0.0/0
- **Outbound:** All allowed

### Public + Private Subnets
- **Public SG:**
  - Inbound: HTTP/HTTPS from internet, SSH/PING from private subnets
  - Outbound: All allowed
- **Private SG:**
  - Inbound: All traffic from public subnets, self-to-self
  - Outbound: All allowed

---

## CIDR Allocation Reference

### Quick CIDR Lookup

```
10.0.0.0/16   - vpc-single-private-subnet (10.0.1.0/24 private)
10.1.0.0/16   - vpc-single-public-subnet (10.1.1.0/24 public)
10.2.0.0/16   - vpc-public-private-subnets (10.2.1.0/24 pub, 10.2.2.0/24 priv)
10.3.0.0/16   - vpc-two-public-private-subnets (10.3.1-2.0/24 pub, 10.3.3-4.0/24 priv)
10.4.0.0/16   - vpc-two-public-four-private-subnets (10.4.1-2.0/24 pub, 10.4.3-6.0/24 priv)
```

---

## Next Steps

1. Choose a parameter file matching your use case
2. Upload templates to S3: `aws s3 sync templates/ s3://your-bucket/templates/`
3. Deploy: `aws cloudformation create-stack --template-body file://templates/parent-stack.yaml --parameters file://parameters/vpc-xxx.json`
4. Verify outputs: `aws cloudformation describe-stacks --stack-name your-stack-name`
