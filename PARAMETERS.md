# Parameter Files Guide

This document explains each parameter file and the VPC architectures they represent.

## Overview of Parameter Files

All parameter files use **non-overlapping CIDR ranges** to allow simultaneous deployment of multiple stacks for testing:

| File | App Prefix | VPC CIDR | VPCs | Public Subnets | Private Subnets | NAT Gateway | Use Case |
|------|-----------|----------|------|-----------------|-----------------|-------------|----------|
| `vpc-networking-vpc-only.json` | `core` | `10.0.0.0/16` | 1 | 0 | 1 | ❌ | Minimal VPC setup |
| `vpc-networking-private-only.json` | `isolated` | `10.0.0.0/16` | 1 | 0 | 1 | ❌ | Database/Backend tier |
| `vpc-networking-public-only.json` | `web` | `10.0.0.0/16` | 1 | 1 | 1 | ❌ | Web/Public tier only |
| `vpc-networking-private-public-subnates.json` | `app` | `10.0.0.0/16` | 1 | 1 | 1 | ✅ | Multi-tier application (Production-Ready) |

## Detailed Architecture Breakdown

### 1. `vpc-networking-vpc-only.json`

**App Prefix:** `core`

**Architecture:**
```
VPC: 10.0.0.0/16
├── DNS enabled
└── Private Subnet 1: 10.0.10.0/24 (us-east-1a)
    └── No internet access
    └── No NAT Gateway
    └── No NACLs
```

**Use Case:** 
- Minimal VPC setup
- Foundation for testing
- Bare minimum infrastructure

**Key Resources Created:**
- 1 VPC
- 1 Private Subnet
- 1 Private Route Table
- 2 Security Groups (public + private)

**Deployment:**
```bash
aws cloudformation create-stack \
  --stack-name core-vpc \
  --template-body file://templates/vpc-networking.yaml \
  --parameters file://parameters/vpc-networking-vpc-only.json \
  --region us-east-1
```

---

### 2. `vpc-networking-private-only.json`

**App Prefix:** `isolated`

**Architecture:**
```
VPC: 10.0.0.0/16
├── DNS enabled
├── Network ACL (enhanced security)
└── Private Subnet 1: 10.0.10.0/24 (us-east-1a)
    ├── No internet access
    ├── No NAT Gateway
    └── Associated with NACL
```

**Use Case:**
- Database tier (isolated, no internet)
- Backend data stores
- Security-sensitive workloads

**Key Resources Created:**
- 1 VPC
- 1 Private Subnet
- 1 Private Route Table
- 1 Network ACL (with inbound/outbound rules)
- 2 Security Groups

**Deployment:**
```bash
aws cloudformation create-stack \
  --stack-name isolated-vpc \
  --template-body file://templates/vpc-networking.yaml \
  --parameters file://parameters/vpc-networking-private-only.json \
  --region us-east-1
```

---

### 3. `vpc-networking-public-only.json`

**App Prefix:** `web`

**Architecture:**
```
VPC: 10.0.0.0/16
├── Internet Gateway
├── DNS enabled
├── Public Subnet 1: 10.0.1.0/24 (us-east-1a)
│   ├── Auto-assign public IPs: Yes
│   ├── Route: 0.0.0.0/0 → IGW
│   └── Network ACL enabled
└── Private Subnet 1: 10.0.10.0/24 (us-east-1a)
    ├── No NAT Gateway
    └── Network ACL enabled
```

**Use Case:**
- Web/public tier only
- Load balancers and web servers
- No backend internet access (private tier isolated)

**Key Resources Created:**
- 1 VPC
- 1 Public Subnet (with auto-assigned public IPs)
- 1 Private Subnet (isolated, no internet)
- 2 Route Tables (public + private)
- 1 Internet Gateway
- 1 Network ACL
- 2 Security Groups

**Traffic Flow:**
```
Internet → IGW → Public Subnet (web tier)
                        ↕
                   Private Subnet (isolated)
```

**Deployment:**
```bash
aws cloudformation create-stack \
  --stack-name web-vpc \
  --template-body file://templates/vpc-networking.yaml \
  --parameters file://parameters/vpc-networking-public-only.json \
  --region us-east-1
```

---

### 4. `vpc-networking-private-public-subnates.json` ⭐ Production-Ready

**App Prefix:** `app`

**Architecture:**
```
VPC: 10.0.0.0/16
├── Internet Gateway
├── DNS enabled
├── Public Subnet 1: 10.0.1.0/24 (us-east-1a)
│   ├── NAT Gateway 1
│   ├── Auto-assign public IPs: Yes
│   ├── Route: 0.0.0.0/0 → IGW
│   └── Network ACL enabled
└── Private Subnet 1: 10.0.10.0/24 (us-east-1a)
    ├── Route: 0.0.0.0/0 → NAT Gateway 1 (internet egress via NAT)
    └── Network ACL enabled
```

**Use Case:**
- **Multi-tier applications** (web + backend)
- **Production-ready** single-AZ setup
- Backend services need internet access (updates, API calls)
- ⭐ Recommended for most applications

**Key Resources Created:**
- 1 VPC
- 1 Public Subnet (web tier)
- 1 Private Subnet (app/backend tier)
- 2 Route Tables (public + private)
- 1 Internet Gateway
- 1 NAT Gateway (single, not HA)
- 1 Elastic IP (for NAT Gateway)
- 1 Network ACL
- 2 Security Groups

**Traffic Flow:**
```
Internet
  ↓
IGW
  ↓
Public Subnet (web tier)
  ↕
Private Subnet (app tier)
  ↓
NAT Gateway → Internet (for updates/APIs)
```

**Security Groups:**
- **Public SG:**
  - Inbound: HTTP (80), HTTPS (443) from internet
  - Outbound: All allowed
- **Private SG:**
  - Inbound: All traffic from public subnet, self-to-self
  - Outbound: All allowed

**Deployment:**
```bash
aws cloudformation create-stack \
  --stack-name app-vpc \
  --template-body file://templates/vpc-networking.yaml \
  --parameters file://parameters/vpc-networking-private-public-subnates.json \
  --region us-east-1
```

**Cost Estimate (Single NAT):**
- NAT Gateway: $0.32/hour × 730 hours/month = ~$235/month
- Data processing: $0.06/GB (variable)

---

## Using Parameter Files

### Deploy a Single Stack

**Deploy the production-ready multi-tier application:**
```bash
aws cloudformation create-stack \
  --stack-name app-vpc-stack \
  --template-body file://templates/vpc-networking.yaml \
  --parameters file://parameters/vpc-networking-private-public-subnates.json \
  --region us-east-1
```

**Deploy an isolated database VPC:**
```bash
aws cloudformation create-stack \
  --stack-name isolated-db-vpc \
  --template-body file://templates/vpc-networking.yaml \
  --parameters file://parameters/vpc-networking-private-only.json \
  --region us-east-1
```

### Deploy Multiple Stacks Simultaneously

Deploy all configurations for comparison (all use 10.0.0.0/16, so deploy sequentially or in different regions):

```bash
for param_file in parameters/vpc-networking-*.json; do
  stack_name=$(basename "$param_file" .json)
  aws cloudformation create-stack \
    --stack-name "$stack_name" \
    --template-body file://templates/vpc-networking.yaml \
    --parameters file://"$param_file" \
    --region us-east-1
done
```

### Update a Stack

```bash
# Update from public-only to app stack (add NAT Gateway)
aws cloudformation update-stack \
  --stack-name web-vpc \
  --template-body file://templates/vpc-networking.yaml \
  --parameters file://parameters/vpc-networking-private-public-subnates.json \
  --region us-east-1
```

### Monitor Stack Events

```bash
aws cloudformation describe-stack-events \
  --stack-name app-vpc-stack \
  --region us-east-1 \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

### Retrieve Stack Outputs

```bash
# Get all VPC details
aws cloudformation describe-stacks \
  --stack-name app-vpc-stack \
  --region us-east-1 \
  --query 'Stacks[0].Outputs'
```

---

## Quick Configuration Selection Guide

| Need | Configuration | App Prefix |
|------|---------------|-----------|
| Just a VPC with minimal setup | `vpc-networking-vpc-only.json` | `core` |
| Isolated database/backend tier | `vpc-networking-private-only.json` | `isolated` |
| Web/public tier only | `vpc-networking-public-only.json` | `web` |
| **Multi-tier application (Recommended)** | `vpc-networking-private-public-subnates.json` | `app` |

---

## Security Group Rules by Configuration

### Public-Only Configuration (web)
- **Public SG:**
  - Inbound: HTTP (80), HTTPS (443) from 0.0.0.0/0
  - Outbound: All allowed
- **Private SG:**
  - Inbound: All traffic from public SG, self-to-self
  - Outbound: All allowed

### Multi-Tier Application (app) ⭐
- **Public SG (Web Tier):**
  - Inbound: HTTP (80), HTTPS (443) from 0.0.0.0/0
  - Outbound: All allowed
- **Private SG (App Tier):**
  - Inbound: All traffic from public SG (10.0.1.0/24), self-to-self
  - Outbound: All allowed (via NAT Gateway to internet)

### Private-Only Configuration (isolated)
- **Public SG:**
  - Inbound: None (unused)
  - Outbound: All allowed
- **Private SG:**
  - Inbound: Self-to-self only
  - Outbound: All allowed (no internet access)

---

## CIDR Allocation Reference

### Subnet Details by Configuration

```
vpc-networking-vpc-only.json (core)
└── VPC CIDR: 10.0.0.0/16
    └── Private Subnet: 10.0.10.0/24 (us-east-1a)

vpc-networking-private-only.json (isolated)
└── VPC CIDR: 10.0.0.0/16
    └── Private Subnet: 10.0.10.0/24 (us-east-1a)
    └── NACL: Enabled

vpc-networking-public-only.json (web)
└── VPC CIDR: 10.0.0.0/16
    ├── Public Subnet: 10.0.1.0/24 (us-east-1a)
    └── Private Subnet: 10.0.10.0/24 (us-east-1a)

vpc-networking-private-public-subnates.json (app) ⭐
└── VPC CIDR: 10.0.0.0/16
    ├── Public Subnet: 10.0.1.0/24 (us-east-1a)
    │   └── NAT Gateway: Elastic IP allocation
    └── Private Subnet: 10.0.10.0/24 (us-east-1a)
        └── Route via NAT Gateway for internet egress
```

---

## Summary Table: VPC & Subnet Count by Configuration

| Configuration | App Prefix | Total VPCs | Public Subnets | Private Subnets | Internet Gateway | NAT Gateway | Route Tables | NACLs | Security Groups |
|---|---|---|---|---|---|---|---|---|---|
| vpc-networking-vpc-only | core | 1 | 0 | 1 | ❌ | 0 | 1 | ❌ | 2 |
| vpc-networking-private-only | isolated | 1 | 0 | 1 | ❌ | 0 | 1 | ✅ | 2 |
| vpc-networking-public-only | web | 1 | 1 | 1 | ✅ | 0 | 2 | ✅ | 2 |
| vpc-networking-private-public-subnates | app | 1 | 1 | 1 | ✅ | 1 | 2 | ✅ | 2 |

**Summary Totals (All 4 configurations combined):**
- Total VPCs: **4**
- Total Public Subnets: **3**
- Total Private Subnets: **4**
- Total NAT Gateways: **1**
- Total Internet Gateways: **3**

---

## Next Steps

1. **Choose a parameter file** based on your architecture needs:
   - **Production multi-tier app?** → Use `vpc-networking-private-public-subnates.json` (app)
   - **Isolated database?** → Use `vpc-networking-private-only.json` (isolated)
   - **Public web tier?** → Use `vpc-networking-public-only.json` (web)
   - **Minimal setup?** → Use `vpc-networking-vpc-only.json` (core)

2. **Customize parameters** if needed:
   - Modify VPC CIDR, subnet CIDR, AZs in the JSON file
   - Update `ApplicationPrefix` for resource naming
   - Adjust `EnvironmentName` (dev/staging/prod)

3. **Deploy the stack:**
   ```bash
   aws cloudformation create-stack \
     --stack-name my-vpc-stack \
     --template-body file://templates/vpc-networking.yaml \
     --parameters file://parameters/vpc-networking-private-public-subnates.json \
     --region us-east-1
   ```

4. **Verify deployment:**
   ```bash
   aws cloudformation describe-stacks \
     --stack-name my-vpc-stack \
     --region us-east-1 \
     --query 'Stacks[0].{Status:StackStatus,Outputs:Outputs}'
   ```

5. **Monitor events** (if needed):
   ```bash
   aws cloudformation describe-stack-events \
     --stack-name my-vpc-stack \
     --region us-east-1
   ```
