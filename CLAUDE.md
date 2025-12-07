# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository contains reusable AWS CloudFormation templates for deploying static website infrastructure with CloudFront CDN, S3 origin, SSL/TLS certificates, and VPC networking.

## Template Architecture

### Stack Dependency Order

The templates have a specific deployment order due to dependencies:

1. **acm_certificate.yaml** - MUST be deployed in `us-east-1` region (CloudFront requirement)
2. **cdn.yaml** - References the certificate ARN from step 1
3. **vpc.yaml** - Optional, independent stack for backend infrastructure

### Cross-Stack References

- `acm_certificate.yaml` exports `CertificateArn` as `{Name}-certificate-{Stage}`
- `cdn.yaml` requires `AcmCertificateArn` parameter from the certificate stack
- Templates use resource suffixes (Name + Stage) for multi-environment deployments

### Key Design Patterns

**ACM Certificate (acm_certificate.yaml):**
- Supports two modes via `CertificateMode` parameter: `single` (specific subdomain) or `wildcard` (*.domain.com)
- Uses conditional logic to create different certificate configurations
- DNS validation records automatically created in Route53
- Optional root domain as Subject Alternative Name via `IncludeRootDomain`

**CDN (cdn.yaml):**
- CloudFront Origin Access Identity (OAI) ensures S3 bucket is not publicly accessible
- Uses regional S3 endpoint format for origin (not website endpoint)
- Default root object set to `index.html` for SPA support
- Creates both A (IPv4) and AAAA (IPv6) Route53 records
- Conditional resources for root domain support via `IncludeRootDomain` parameter
- CloudFront distribution has `Retain` deletion policy to prevent accidental deletion

**VPC (vpc.yaml):**
- Scalable design supporting 1-3 availability zones via `AvailabilityZoneCount` parameter
- Uses conditions (`Has2AZ`, `Has3AZ`) to conditionally create resources
- One NAT Gateway per AZ for high availability (incurs hourly charges)
- S3 VPC Endpoint for private S3 access without internet gateway

## Common Commands

### Deploy Certificate (CRITICAL: us-east-1 only)

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

### Get Certificate ARN

```bash
aws cloudformation describe-stacks \
  --stack-name my-domain-cert \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`CertificateArn`].OutputValue' \
  --output text
```

### Deploy CDN

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

### Deploy VPC

```bash
aws cloudformation deploy \
  --template-file vpc.yaml \
  --stack-name my-vpc \
  --parameter-overrides \
    CidrBlock=10.0.0.0/16 \
    AvailabilityZoneCount=2
```

### Validate Templates

```bash
aws cloudformation validate-template --template-body file://acm_certificate.yaml
aws cloudformation validate-template --template-body file://cdn.yaml
aws cloudformation validate-template --template-body file://vpc.yaml
```

### Get Stack Outputs

```bash
# Get all outputs from a stack
aws cloudformation describe-stacks --stack-name <stack-name> \
  --query 'Stacks[0].Outputs' --output table

# Get specific output value
aws cloudformation describe-stacks --stack-name <stack-name> \
  --query 'Stacks[0].Outputs[?OutputKey==`<OutputKey>`].OutputValue' \
  --output text
```

### Upload Content to CDN

```bash
# Get bucket name from stack outputs
BUCKET_NAME=$(aws cloudformation describe-stacks \
  --stack-name my-cdn \
  --query 'Stacks[0].Outputs[?OutputKey==`ContentBucket`].OutputValue' \
  --output text)

# Sync content to bucket
aws s3 sync ./my-website/ s3://$BUCKET_NAME/
```

## AWS CLI Configuration

The repository is configured for use with:
- AWS Region: `us-east-2` (but ACM cert MUST be in `us-east-1`)
- AWS Profile: `cuppett`

Local Claude permissions (`.claude/settings.local.json`) allow:
- CloudFormation operations
- ACM certificate description
- CloudFront operations

## Important Constraints

1. ACM certificates for CloudFront MUST be created in `us-east-1` region
2. The CloudFront distribution uses a `Retain` deletion policy - it will NOT be deleted when the stack is deleted
3. All templates use AES256 server-side encryption for S3 buckets
4. CloudFront enforces HTTPS-only with TLS 1.2 minimum
5. NAT Gateways in VPC template incur hourly AWS charges
6. S3 buckets use Origin Access Identity - they are NOT publicly accessible
