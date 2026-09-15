# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **CloudFormation nested stack template** repository that provides reusable templates for deploying highly available private subnets across multiple AWS Availability Zones. The template supports multi-AZ NAT Gateways, granular routing, and Network Access Control Lists (NACLs) for secure private subnet architecture.

**Key characteristics:**

- Nested CloudFormation template (referenced via `TemplateURL`)
- Multi-AZ support: Creates 1-4 private subnets across Availability Zones
- One route table per subnet for independent AZ-level routing control
- Optional multi-AZ NAT Gateways with Elastic IPs for internet egress
- Optional Network ACLs with stateless inbound/outbound rules
- Parameterized configuration for flexible deployments
- Exports outputs for cross-stack references

## Project Structure

```text
templates/
└── vpc-subnets.yaml                # Nested template: Private subnets with multi-AZ NAT

parameters/
└── parameters.json                 # Parameter values (template for actual deployments)

.github/workflows/
├── ci.yaml                         # Validates, deploys, and cleans up templates
├── release.yaml                    # Semantic release on push to main
├── setup-environments.yaml         # GitHub environment setup
├── notify.yaml                     # CI notifications
├── create-branch.yaml              # Auto-create feature branches from issues
├── claude.yaml                     # Claude integration
└── claude-code-review.yaml         # Code review workflow

scripts/plugins/
├── release.config.js               # Semantic-release configuration
└── (other release plugins)

.claude/
├── settings.json                   # Claude Code workspace settings
└── settings.local.json             # Local overrides

.devcontainer/
└── devcontainer.json               # Dev container setup (Node.js 20)

package.json                        # Dependencies: semantic-release, commitizen
README.md                           # Template documentation
```

## Development Commands

### Install dependencies

```bash
npm ci
```

### Trigger semantic release (usually automatic on main)

```bash
npm run release
```

### Commit with conventional commit format

```bash
npx cz commit
```

Select `feat`, `fix`, or `chore` type. Only `feat` and `fix` trigger releases.

## Architecture: Private Subnets with Multi-AZ NAT

### High-Level Design

This template creates a private subnet tier for VPCs, supporting:

1. **Private Subnets (1-4 across AZs)** — No direct internet access; instances use NAT for egress
2. **Route Tables (One per AZ)** — Enables independent routing policies per Availability Zone
3. **Multi-AZ NAT Gateways** — Optional HA setup with one NAT Gateway per AZ
4. **Network ACLs** — Optional stateless firewall rules for subnet isolation

### Why One Route Table Per Subnet?

In AWS, a **route table must be associated with exactly one or more subnets**, but **subnets can reference only one route table**. This design creates independent routing per AZ, enabling:

- Different NAT Gateways in different AZs (for NAT HA)
- Fine-grained routing policies per availability zone
- Predictable traffic paths for compliance/audit

### Multi-AZ NAT Gateway Pattern

The template supports two NAT configurations:

**Single NAT (Cost-Optimized):**
- One NAT Gateway in the first public subnet
- All private subnets (all AZs) route through it
- Lower cost, single point of failure

**Multi-AZ NAT (High Availability):**
- One NAT Gateway per Availability Zone
- Private subnet routes to the NAT in its own AZ
- Avoids cross-AZ data transfer charges
- No dependency between AZs for internet egress
- Recommended for production

### Network ACLs (Optional)

NACLs provide stateless network filtering:
- **Inbound rules**: Allow traffic from VPC CIDR and ephemeral ports from internet
- **Outbound rules**: Allow all outbound traffic
- Can be customized for stricter policies if needed

## Key Files to Understand

### `templates/vpc-subnets.yaml`

**Purpose:** Creates private subnets with NAT Gateways and Network ACLs

**Key Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `EnvironmentName` | String | Yes | Environment name prefix (e.g., `prod`, `staging`, `dev`) |
| `VPCId` | AWS::EC2::VPC::Id | Yes | Existing VPC to attach subnets to |
| `SubnetCount` | Number | Yes | Number of private subnets (1-4, one per AZ) |
| `SubnetCidrBlocks` | CommaDelimitedList | Yes | CIDR blocks for subnets (e.g., `10.0.1.0/24,10.0.2.0/24`) |
| `AvailabilityZones` | CommaDelimitedList | Yes | AZs for subnets (e.g., `us-east-1a,us-east-1b,us-east-1c`) |
| `EnableNATGateway` | String | Yes | Enable NAT Gateway for private subnet internet egress (`true`/`false`) |
| `EnableHighAvailabilityNAT` | String | Yes | Create one NAT Gateway per AZ (`true`/`false`); requires `EnableNATGateway=true` |
| `PublicSubnetIds` | CommaDelimitedList | Yes | Public subnet IDs where NAT Gateways will be placed (must match subnet count) |
| `EnableNACLs` | String | No | Enable Network ACLs (default: `true`) |

**Key Outputs:**

- `SubnetIds` — Private subnet IDs (comma-separated)
- `RouteTableIds` — Route table IDs (comma-separated)
- `NATGatewayIds` — NAT Gateway IDs (if enabled)
- `NATGatewayIps` — Elastic IPs for NAT Gateways (if enabled)
- `NetworkAclId` — Network ACL ID (if enabled)
- `SubnetCount` — Number of subnets created

**Conditions (Nested Logic):**

The template uses AWS CloudFormation conditions to conditionally create resources:

- `CreatePrivateSubnet1-4` — Create 1-4 subnets based on `SubnetCount`
- `EnableNAT` — NAT Gateway enabled
- `EnableHANAT` — Both NAT and HA NAT enabled (creates one NAT per AZ)
- `CreateNATGateway1-4` — Create 1-4 NAT Gateways based on conditions
- `CreateNACLs` — Network ACL enabled

This pattern avoids hard-coding subnet/NAT counts; CloudFormation conditionally creates resources as needed.

## Testing & Validation

### Local Template Validation

Validate syntax before deployment:

```bash
aws cloudformation validate-template \
  --template-body file://templates/vpc-subnets.yaml
```

### Example: Deploy 2-AZ Private Subnet Stack

**Step 1: Prepare parameters**

Create a parameter file (`parameters/prod-2az.json`):

```json
[
  {
    "ParameterKey": "EnvironmentName",
    "ParameterValue": "prod"
  },
  {
    "ParameterKey": "VPCId",
    "ParameterValue": "vpc-0123456789abcdef0"
  },
  {
    "ParameterKey": "SubnetCount",
    "ParameterValue": "2"
  },
  {
    "ParameterKey": "SubnetCidrBlocks",
    "ParameterValue": "10.0.1.0/24,10.0.2.0/24"
  },
  {
    "ParameterKey": "AvailabilityZones",
    "ParameterValue": "us-east-1a,us-east-1b"
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
    "ParameterKey": "PublicSubnetIds",
    "ParameterValue": "subnet-public-1a,subnet-public-1b"
  },
  {
    "ParameterKey": "EnableNACLs",
    "ParameterValue": "true"
  }
]
```

**Step 2: Deploy the stack**

```bash
aws cloudformation create-stack \
  --stack-name prod-private-subnets \
  --template-body file://templates/vpc-subnets.yaml \
  --parameters file://parameters/prod-2az.json \
  --region us-east-1
```

**Step 3: Monitor stack creation**

```bash
aws cloudformation wait stack-create-complete \
  --stack-name prod-private-subnets \
  --region us-east-1

aws cloudformation describe-stacks \
  --stack-name prod-private-subnets \
  --region us-east-1 \
  --query 'Stacks[0].Outputs'
```

### Example: Update Stack (Enable NAT after initial creation)

If you initially deployed without NAT and want to add it:

```bash
aws cloudformation update-stack \
  --stack-name prod-private-subnets \
  --template-body file://templates/vpc-subnets.yaml \
  --parameters file://parameters/prod-2az-with-nat.json \
  --region us-east-1
```

### Cleanup

```bash
aws cloudformation delete-stack \
  --stack-name prod-private-subnets \
  --region us-east-1
```

## AWS Credentials & Environment Variables

### GitHub CI Environment

The CI workflow (`.github/workflows/ci.yaml`) requires these GitHub environment variables:

- `AWS_REGION`: CloudFormation deployment region
- `AWS_ACCOUNT_ID`: AWS account to deploy into
- `OIDC_ROLE_NAME`: IAM role name for OIDC trust
- `CFN_TEMPLATES_S3_BUCKET`: S3 bucket where templates are stored (for CI deployments)

### Local AWS Credentials

For local testing, ensure AWS credentials are configured:

```bash
aws configure
# or set AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_DEFAULT_REGION
```

## Conventional Commits & Release Flow

This repo enforces conventional commits to drive semantic versioning:

```bash
npx cz commit
```

Commit types:

- `feat: add support for X` → triggers MINOR release
- `fix: correct behavior of Y` → triggers PATCH release
- `chore: update deps` → no release
- `docs: clarify README` → no release

Only commits to `main` trigger releases. Feature branches use this format but releases happen on merge to main.

## When Modifying Templates

1. **Edit the template YAML** in `templates/vpc-subnets.yaml`
2. **Create parameter files** in `parameters/` for each environment/configuration
3. **Test locally** with `aws cloudformation validate-template`
4. **Create a PR** with conventional commit message (e.g., `feat: add support for 4-AZ deployments`)
5. **CI validates and deploys** to CI environment automatically
6. **Merge to main** → release workflow creates version tag and GitHub release

## Common Development Tasks

### Add Support for a New Parameter

1. Add to `Parameters` section in `vpc-subnets.yaml`
2. Use the parameter in resource properties or conditions
3. Add to parameter file examples in `parameters/`
4. Update this CLAUDE.md with parameter docs
5. Test with `aws cloudformation validate-template`
6. Commit: `feat: add {parameter name} parameter`

### Modify NAT Gateway Behavior

The template currently uses conditions to control NAT creation:
- Set `EnableNATGateway=true/false` to enable/disable NAT entirely
- Set `EnableHighAvailabilityNAT=true/false` to control multi-AZ setup

To add new NAT features (e.g., source IP logging), add resource properties to the `NATGateway1-4` resources and update `Outputs`.

### Customize Network ACL Rules

NACL rules are hardcoded in the template (RuleNumbers 100, 110, etc.). To change them:
1. Edit `PrivateNACLInboundFromPublic`, `PrivateNACLInboundEphemeral`, `PrivateNACLOutboundAll`
2. Adjust `CidrBlock`, `PortRange`, and `RuleNumber` as needed
3. Consider adding parameters instead of hardcoding for flexibility
4. Test with a staging deployment before production

## Dev Container

Pre-configured with:

- Node.js 20
- GitHub Copilot extension

Use via VS Code: `code --remote-container-url <repo-url>`

## Current Branch Structure

Main branch is the release branch. Feature work branches from here and merges back via PR. Branch naming follows: `{type}/CFN-{issue-number}-{slug}` (e.g., `feature/CFN-42-add-ha-nat`).

## Deployment Patterns

### Pattern 1: Single NAT (Development)

```yaml
EnableNATGateway: "true"
EnableHighAvailabilityNAT: "false"  # Single NAT in first AZ
```

**Use case:** Dev/test environments where cost is a priority

### Pattern 2: Multi-AZ NAT (Production)

```yaml
EnableNATGateway: "true"
EnableHighAvailabilityNAT: "true"   # One NAT per AZ
SubnetCount: 3
```

**Use case:** Production with redundancy and no cross-AZ data transfer costs

### Pattern 3: Private with No Internet Egress

```yaml
EnableNATGateway: "false"           # No NAT Gateways
EnableNACLs: "true"                 # NACLs still provide isolation
```

**Use case:** Airgapped environments or isolated subnets for databases
