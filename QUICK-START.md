# Quick Start Guide

## TL;DR — Deploy in 3 Steps

### 1. Upload Templates to S3

```bash
aws s3 sync templates/ s3://YOUR-BUCKET/templates/
```

### 2. Choose a Parameter File

```bash
ls parameters/vpc-*.json

# Pick one:
# - vpc-single-private-subnet.json      (Dev, isolated)
# - vpc-single-public-subnet.json       (Public services)
# - vpc-public-private-subnets.json     (Standard 2-tier)
# - vpc-two-public-private-subnets.json (Multi-AZ)
# - vpc-two-public-four-private-subnets.json (Production HA)
```

### 3. Deploy

```bash
aws cloudformation create-stack \
  --stack-name my-vpc \
  --template-body file://templates/parent-stack.yaml \
  --parameters file://parameters/vpc-two-public-four-private-subnets.json \
  --region us-east-1
```

### Monitor Deployment

```bash
aws cloudformation describe-stacks --stack-name my-vpc --query 'Stacks[0].StackStatus'

# Wait for completion
aws cloudformation wait stack-create-complete --stack-name my-vpc

# View outputs
aws cloudformation describe-stacks --stack-name my-vpc --query 'Stacks[0].Outputs'
```

---

## What Each Parameter File Creates

| Parameter File | VPC CIDR | Subnets | NAT | Best For |
|---|---|---|---|---|
| `vpc-single-private-subnet.json` | 10.0.0.0/16 | 1 private | ❌ No | Isolated workloads |
| `vpc-single-public-subnet.json` | 10.1.0.0/16 | 1 public | ❌ No | Public services only |
| `vpc-public-private-subnets.json` | 10.2.0.0/16 | 1 pub + 1 priv | ✅ Single | Dev/test 2-tier app |
| `vpc-two-public-private-subnets.json` | 10.3.0.0/16 | 2 pub + 2 priv | ✅ Single | Multi-AZ, cost-optimized |
| `vpc-two-public-four-private-subnets.json` | 10.4.0.0/16 | 2 pub + 4 priv | ✅ HA | **Production HA** |

---

## Cleanup

```bash
# Delete stack (all nested stacks delete automatically)
aws cloudformation delete-stack --stack-name my-vpc

# Wait for deletion
aws cloudformation wait stack-delete-complete --stack-name my-vpc
```

---

## Documentation Files

| File | Purpose |
|------|---------|
| **CLAUDE.md** | Comprehensive dev guide & architecture |
| **PARAMETERS.md** | Detailed explanation of each parameter file |
| **IMPLEMENTATION-SUMMARY.md** | Full implementation details & examples |
| **QUICK-START.md** | This file — quick reference |

---

## Common Customizations

### Change CIDR Blocks

Edit the parameter JSON file and adjust:
- `VPCCidrBlock`: VPC CIDR (e.g., `10.0.0.0/16`)
- `PublicSubnetCidrBlocks`: Comma-delimited list (e.g., `10.0.1.0/24,10.0.2.0/24`)
- `PrivateSubnetCidrBlocks`: Comma-delimited list

### Enable/Disable Internet Gateway

In parameter JSON:
```json
"EnableInternetGateway": "true"  // or "false"
```

### Enable Multi-AZ NAT

In parameter JSON:
```json
"EnableHighAvailabilityNAT": "true"  // One NAT per AZ
```

### Control Security Group Rules

In parameter JSON:
```json
"AllowSSHFromPrivate": "true",
"AllowPingFromPrivate": "true",
"AllowHTTPFromInternet": "true",
"AllowHTTPSFromInternet": "true"
```

---

## Troubleshooting

### Error: "Access Denied" for S3

Upload templates to S3 first:
```bash
aws s3 sync templates/ s3://YOUR-BUCKET/templates/
```

### Error: "Incorrect number of CIDR blocks"

Ensure the count of subnets matches CIDR blocks:
```json
{
  "PrivateSubnetCount": "2",
  "PrivateSubnetCidrBlocks": "10.0.1.0/24,10.0.2.0/24"  // Must have 2
}
```

### Stack Creation Fails

Check events for details:
```bash
aws cloudformation describe-stack-events \
  --stack-name my-vpc \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

---

## Architecture Diagrams

### Single AZ (vpc-public-private-subnets.json)

```
        Internet
            |
            |
        [IGW]
            |
    ┌───────┴───────┐
    |               |
[Public Subnet]  [EC2 Instance]
  10.2.1.0/24       
    |               
 [NAT GW]
    |
    └───────────────┐
                    |
            [Private Subnet]
              10.2.2.0/24
                    |
            [Database/App]
```

### Multi-AZ HA (vpc-two-public-four-private-subnets.json)

```
Internet
    |
   IGW
    |
    ├─────────────────────────────┐
    |                             |
us-east-1a                   us-east-1b
    |                             |
[Public Subnet]          [Public Subnet]
10.4.1.0/24              10.4.2.0/24
    |                             |
 [NAT-1]                      [NAT-2]
    |                             |
    ├──────────────┬──────────────┤
    |              |              |
[Priv-1a]     [Priv-2b]    [Priv-3c]
10.4.3.0/24  10.4.4.0/24  10.4.5.0/24
    |              |              |
[App/DB]      [App/DB]      [App/DB]

    └──────────────┬──────────────┘
    
                   |
              [Priv-4d]
            10.4.6.0/24
            (us-east-1d)
                   |
              [App/DB]
```

---

## Parameter File Template

Create custom architectures using this template:

```json
{
  "EnvironmentName": "my-env",
  "VPCCidrBlock": "10.5.0.0/16",
  "EnableInternetGateway": "true",
  "EnableDnsHostnames": "true",
  
  "PublicSubnetCount": "2",
  "PublicSubnetCidrBlocks": "10.5.1.0/24,10.5.2.0/24",
  "PublicAvailabilityZones": "us-east-1a,us-east-1b",
  
  "PrivateSubnetCount": "2",
  "PrivateSubnetCidrBlocks": "10.5.3.0/24,10.5.4.0/24",
  "PrivateAvailabilityZones": "us-east-1a,us-east-1b",
  
  "EnableNATGateway": "true",
  "EnableHighAvailabilityNAT": "true",
  "PublicSubnetIds": "",
  "EnableNACLs": "true",
  
  "CreateSecurityGroups": "true",
  "PublicSubnetCidr": "10.5.1.0/23",
  "PrivateSubnetCidr": "10.5.3.0/23",
  
  "AllowSSHFromPrivate": "true",
  "AllowPingFromPrivate": "true",
  "AllowHTTPFromInternet": "true",
  "AllowHTTPSFromInternet": "true"
}
```

---

## Useful AWS CLI Commands

### Get Stack Outputs

```bash
aws cloudformation describe-stacks --stack-name my-vpc \
  --query 'Stacks[0].Outputs[*].[OutputKey,OutputValue]' \
  --output table
```

### List All Stacks

```bash
aws cloudformation list-stacks --query 'StackSummaries[?StackStatus!=`DELETE_COMPLETE`].[StackName,StackStatus]' --output table
```

### Update Stack

```bash
aws cloudformation update-stack \
  --stack-name my-vpc \
  --template-body file://templates/parent-stack.yaml \
  --parameters file://parameters/vpc-xxx.json
```

### Delete Stack

```bash
aws cloudformation delete-stack --stack-name my-vpc
```

---

## Next Steps

1. Read **PARAMETERS.md** for detailed architecture explanations
2. Read **CLAUDE.md** for development & modification guide
3. Check **IMPLEMENTATION-SUMMARY.md** for advanced usage
4. Deploy your first stack using the Quick Start above
