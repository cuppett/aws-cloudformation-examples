# AWS CloudFormation Examples

This repository contains reusable AWS CloudFormation templates for deploying static website infrastructure with CloudFront CDN, S3 origin, SSL/TLS certificates, and VPC networking.

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

## Notes

- The CloudFront distribution has a `Retain` deletion policy to prevent accidental deletion
- All S3 content is encrypted at rest with AES256
- CloudFront uses Origin Access Identity - the S3 bucket is not publicly accessible
- NAT Gateways incur hourly charges (~$0.045/hour per AZ)
- Gateway VPC Endpoints (S3, DynamoDB) are free
- Interface VPC Endpoints cost ~$0.01/hour per AZ plus data processing charges
- For fully private architectures, disable NAT gateways and enable SSM endpoints for instance access
- DNS records are created automatically for configured domains
