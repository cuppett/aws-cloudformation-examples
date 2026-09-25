# AWS CloudFormation Examples

This repository contains reusable AWS CloudFormation templates for deploying static website infrastructure with CloudFront CDN, S3 origin, SSL/TLS certificates, VPC networking, and a bastion/NAT instance.

## Templates

### acm_certificate.yaml

Creates a DNS-validated SSL/TLS certificate in AWS Certificate Manager for use with CloudFront distributions.

**Key Features:**
- DNS validation through Route53
- Two modes: `single` (specific subdomain) or `wildcard` (*.domain.com)
- Optional root domain as Subject Alternative Name
- Must be deployed in `us-east-1` region (CloudFront requirement)

**Important Parameters:**
- `DnsZone` - Your Route53 hosted zone (e.g., `example.com`)
- `HostedZoneId` - Route53 Hosted Zone ID
- `CertificateMode` - `single` or `wildcard`
- `Name` - Subdomain prefix (default: `cdn`)
- `IncludeRootDomain` - Include bare domain as SAN (default: `false`)

### cdn.yaml

Sets up a complete content delivery network with S3 origin, CloudFront distribution, and Route53 DNS records.

**Key Features:**
- S3 bucket with server-side encryption (AES256)
- CloudFront Origin Access Identity for secure S3 access
- Custom caching policy with Brotli/Gzip compression
- HTTPS-only with TLS 1.2 minimum
- IPv4 and IPv6 support (A and AAAA records)
- Optional root domain support
- Default root object set to `index.html`

**Important Parameters:**
- `AcmCertificateArn` - Certificate ARN from acm_certificate stack
- `DnsZone` - Your Route53 hosted zone
- `Name` - CDN name prefix (default: `cdn`)
- `Stage` - Environment stage (default: `main`)
- `IncludeRootDomain` - Create records for bare domain (default: `false`)

### vpc.yaml

Creates a production-ready VPC with configurable availability zones (1-3 AZs) and optional VPC endpoints for fully private architectures.

**Key Features:**
- Public and private subnets in each AZ
- Internet Gateway for public subnet access
- Optional NAT Gateways per AZ for private subnet outbound access
- Optional VPC Gateway Endpoints (S3, DynamoDB) - free
- Optional VPC Interface Endpoints for private AWS service access:
  - SSM Session Manager (ssm, ec2messages, ssmmessages)
  - Container services (ECR API, ECR DKR)
  - Core services (CloudWatch Logs, KMS, STS, Secrets Manager)
  - Messaging services (SQS, SNS, CloudWatch Monitoring)
- Properly configured route tables and security groups

**Important Parameters:**
- `VpcCidr` - VPC CIDR block (default: `10.0.0.0/16`)
- `AvailabilityZoneCount` - Number of AZs: 1, 2, or 3 (default: `1`)
- `EnableNatGateways` - Create NAT gateways (default: `true`)
- `EnableS3Endpoint` - S3 gateway endpoint (default: `true`)
- `EnableSsmEndpoints` - SSM for Session Manager access (default: `false`)
- `EnableEcrEndpoints` - ECR for container images (default: `false`)
- Additional endpoint toggles for DynamoDB, Logs, KMS, STS, Secrets Manager, SQS, SNS, Monitoring

### bastion.yaml

Launches a single bastion host into a public subnet, optionally doubling as a NAT instance for private subnets (a low-cost alternative to NAT Gateways).

**Key Features:**
- Amazon Linux 2023 by default, architecture (arm64/x86_64) picked automatically from the instance family; any cloud-init + dnf AMI (Fedora, CentOS Stream, RHEL) can be supplied instead
- Graviton (t4g, m7g/m8g, c7g/c8g), AMD (t3a, m7a/m8a) and Intel (t3, m7i-flex, c7i-flex) instance types
- Automatic updates via `dnf-automatic` (security-only by default) with an automatic reboot when core packages (kernel, glibc, systemd, ...) change
- SSM Session Manager access through an instance role; SSH is optional (blank `AllowedSshCidr` = no inbound SSH)
- Optional Elastic IP, IMDSv2 required, encrypted gp3 root volume
- NAT mode: disables source/dest check, enables IP forwarding + nftables masquerade, and adds `0.0.0.0/0` routes to the given private route tables

**Important Parameters:**
- `InstanceType` - default `t4g.small`
- `VpcId` / `PublicSubnet` - where to launch
- `AllowedSshCidr` - SSH source CIDR (default: blank, SSM only)
- `SshPublicKey` / `Username` / `KeyName` - optional access configuration
- `EnableNat` - act as NAT instance (default: `false`)
- `PrivateRouteTableIds` - route tables to point at the NAT instance
- `AutoUpdateType` - `security` (default) or `default` (all updates)

## Prerequisites

- AWS CLI installed and configured with appropriate profile
- A Route53 hosted zone for your domain
- IAM permissions for CloudFormation, ACM, CloudFront, S3, Route53, VPC, and EC2

## Deployment Guide

### 1. Deploy ACM Certificate

**Important:** The certificate must be created in `us-east-1` for use with CloudFront.

```bash
aws cloudformation deploy \
  --template-file acm_certificate.yaml \
  --stack-name my-domain-cert \
  --region us-east-1 \
  --parameter-overrides \
    DnsZone=example.com \
    HostedZoneId=Z1234567890ABC \
    CertificateMode=wildcard \
    Name=cdn
```

After deployment, retrieve the certificate ARN:

```bash
aws cloudformation describe-stacks \
  --stack-name my-domain-cert \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`CertificateArn`].OutputValue' \
  --output text
```

### 2. Deploy CDN

Use the certificate ARN from the previous step:

```bash
aws cloudformation deploy \
  --template-file cdn.yaml \
  --stack-name my-cdn \
  --parameter-overrides \
    AcmCertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/abc-123 \
    DnsZone=example.com \
    Name=cdn \
    Stage=main \
    IncludeRootDomain=false
```

### 3. Deploy VPC (Optional)

If you need backend infrastructure, deploy the VPC stack:

```bash
# Standard VPC with NAT gateways
aws cloudformation deploy \
  --template-file vpc.yaml \
  --stack-name my-vpc \
  --parameter-overrides \
    VpcCidr=10.0.0.0/16 \
    AvailabilityZoneCount=2

# Fully private VPC with SSM access (no NAT gateways)
aws cloudformation deploy \
  --template-file vpc.yaml \
  --stack-name my-private-vpc \
  --parameter-overrides \
    VpcCidr=10.0.0.0/16 \
    AvailabilityZoneCount=2 \
    EnableNatGateways=false \
    EnableSsmEndpoints=true \
    EnableLogsEndpoint=true \
    EnableKmsEndpoint=true
```

### Deploy Bastion / NAT Instance (Optional)

```bash
# Plain bastion, SSH from one address plus SSM
aws cloudformation deploy \
  --template-file bastion.yaml \
  --stack-name bastion \
  --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND \
  --parameter-overrides \
    VpcId=vpc-0123456789abcdef0 \
    PublicSubnet=subnet-0123456789abcdef0 \
    AllowedSshCidr=203.0.113.10/32 \
    "SshPublicKey=$(cat ~/.ssh/id_ed25519.pub)"

# Bastion + NAT instance replacing NAT gateways
# 1. Remove the NAT gateways (and their default routes) from the VPC stack
aws cloudformation deploy --template-file vpc.yaml --stack-name my-vpc \
  --parameter-overrides EnableNatGateways=false
# 2. Point the private route tables at the bastion
aws cloudformation deploy \
  --template-file bastion.yaml \
  --stack-name bastion \
  --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND \
  --parameter-overrides \
    VpcId=vpc-0123456789abcdef0 \
    PublicSubnet=subnet-0123456789abcdef0 \
    EnableNat=true \
    VpcCidr=10.0.0.0/16 \
    PrivateRouteTableIds=rtb-aaa,rtb-bbb,rtb-ccc
```

`CAPABILITY_AUTO_EXPAND` is required because the template uses the `AWS::LanguageExtensions` transform to create one route per route table. Private subnets lose outbound internet access between steps 1 and 2.

### 4. Upload Content

After the CDN is deployed, upload your static website content:

```bash
# Get the bucket name from stack outputs
BUCKET_NAME=$(aws cloudformation describe-stacks \
  --stack-name my-cdn \
  --query 'Stacks[0].Outputs[?OutputKey==`ContentBucket`].OutputValue' \
  --output text)

# Upload your content
aws s3 sync ./my-website/ s3://$BUCKET_NAME/
```

## Parameters Reference

### acm_certificate.yaml

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `Name` | String | `cdn` | Subdomain prefix |
| `DnsZone` | String | - | Route53 zone name (required) |
| `HostedZoneId` | String | - | Route53 Hosted Zone ID (required) |
| `CertificateMode` | String | `single` | `single` or `wildcard` |
| `IncludeRootDomain` | String | `false` | Include root domain as SAN |

### cdn.yaml

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `Name` | String | `cdn` | CDN name prefix |
| `Stage` | String | `main` | Environment stage |
| `AcmCertificateArn` | String | - | ACM certificate ARN (required) |
| `DnsZone` | String | - | Route53 zone name (required) |
| `IncludeRootDomain` | String | `false` | Create root domain records |

### vpc.yaml

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `VpcCidr` | String | `10.0.0.0/16` | VPC CIDR block |
| `AvailabilityZoneCount` | Number | `1` | Number of AZs (1-3) |
| `SubnetBits` | Number | `12` | Subnet size bits (5-13) |
| `EnableNatGateways` | String | `true` | Create NAT gateways |
| `EnableS3Endpoint` | String | `true` | S3 gateway endpoint (free) |
| `EnableDynamoDbEndpoint` | String | `false` | DynamoDB gateway endpoint (free) |
| `EnableSsmEndpoints` | String | `false` | SSM endpoints for Session Manager |
| `EnableEcrEndpoints` | String | `false` | ECR endpoints for containers |
| `EnableLogsEndpoint` | String | `false` | CloudWatch Logs endpoint |
| `EnableKmsEndpoint` | String | `false` | KMS endpoint |
| `EnableStsEndpoint` | String | `false` | STS endpoint |
| `EnableSecretsManagerEndpoint` | String | `false` | Secrets Manager endpoint |
| `EnableSqsEndpoint` | String | `false` | SQS endpoint |
| `EnableSnsEndpoint` | String | `false` | SNS endpoint |
| `EnableMonitoringEndpoint` | String | `false` | CloudWatch Monitoring endpoint |

### bastion.yaml

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `Name` | String | `bastion` | Name tag for created resources |
| `InstanceType` | String | `t4g.small` | Instance type (t4g/t3a/t3, m7g/m8g/c7g/c8g, m7a/m8a, m7i-flex/c7i-flex) |
| `ImageId` | String | - | AMI override; blank = latest Amazon Linux 2023 |
| `AutoUpdateType` | String | `security` | `security` or `default` (all updates) |
| `Username` | String | - | Extra sudo user to create |
| `SshPublicKey` | String | - | SSH public key for `Username` or the default user |
| `KeyName` | String | - | Existing EC2 key pair |
| `AllowedSshCidr` | String | - | SSH source CIDR; blank = no inbound SSH |
| `VpcId` | VPC ID | - | VPC (required) |
| `PublicSubnet` | Subnet ID | - | Public subnet (required) |
| `AllocateElasticIp` | String | `true` | Attach an Elastic IP |
| `EnableNat` | String | `false` | Act as a NAT instance |
| `VpcCidr` | String | `10.0.0.0/16` | CIDR allowed through the NAT |
| `PrivateRouteTableIds` | List | - | Route tables to send `0.0.0.0/0` to the NAT |

## Stack Outputs

### acm_certificate.yaml

- `CertificateArn` - ARN of the created certificate (exported as `{Name}-certificate-{Stage}`)

### cdn.yaml

- `ContentBucket` - Name of the S3 content bucket
- `CdnUrl` - HTTPS URL of the CloudFront distribution

### vpc.yaml

- `VpcId` - VPC ID
- `PublicSubnetIds` - Comma-separated public subnet IDs
- `PrivateSubnetIds` - Comma-separated private subnet IDs
- `PublicRouteTableId` - Public route table ID
- `PrivateRouteTableIds` - Comma-separated private route table IDs

### bastion.yaml

- `BastionInstanceId` - Instance ID
- `BastionPublicIp` - Public IP (Elastic IP when allocated)
- `BastionPrivateIp` - Private IP
- `BastionSecurityGroupId` - Security group ID
- `SsmSessionCommand` - Ready-to-run `aws ssm start-session` command

## Notes

- The CloudFront distribution has a `Retain` deletion policy to prevent accidental deletion
- All S3 content is encrypted at rest with AES256
- CloudFront uses Origin Access Identity - the S3 bucket is not publicly accessible
- NAT Gateways incur hourly charges (~$0.045/hour per AZ)
- Gateway VPC Endpoints (S3, DynamoDB) are free
- Interface VPC Endpoints cost ~$0.01/hour per AZ plus data processing charges
- For fully private architectures, disable NAT gateways and enable SSM endpoints for instance access
- DNS records are created automatically for configured domains
- A NAT instance is a single point of failure for private subnet egress and is interrupted by automatic update reboots; use NAT Gateways where that matters
