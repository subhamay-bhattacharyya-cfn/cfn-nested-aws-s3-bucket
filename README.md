# CloudFormation Template: S3 Bucket

<!-- Row 1: Status - Most Important -->
[![Release](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)&nbsp;[![GitHub Repo](https://img.shields.io/badge/GitHub-Repository-blue?logo=github)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/github/languages/top/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazon&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-s3-bucket/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/bc68df66aeb973d594261ab6d6c45567/raw/cfn-nested-aws-s3-bucket.json)](https://gist.github.com/subhamay-bhattacharyya/bc68df66aeb973d594261ab6d6c45567)

This repository contains a reusable nested CloudFormation template for deploying S3 buckets with enterprise-grade security best practices and flexible configuration options.

## Overview

This is a **nested stack template** designed to be invoked from a parent/root CloudFormation stack. The template is highly parameterized, supports multiple deployment scenarios, and includes comprehensive security controls.

## Template Files

### CloudFormation Templates

- **`cloudformation/template.yaml`** — Complete S3 bucket template with encryption, logging, versioning, website hosting, and access logging

### Parameter Files

- **`cloudformation/parameters.json`** — Key-value parameter file for deployment configuration

## Template Features

### Core Features

- ✅ **Versioning** — Always enabled for data protection and recovery
- ✅ **Public Access Blocking** — All four options enabled by default
- ✅ **Smart Bucket Naming** — Automatic naming with project prefix, account ID, environment, and region
- ✅ **Resource Tagging** — Automatic tags for ProjectName and Environment

### Security Features

- ✅ **Dual Encryption Support** — SSE-S3 (default) or SSE-KMS (optional with flexible key formats)
- ✅ **KMS Key Flexibility** — Accepts key name, alias, or full ARN for maximum compatibility
- ✅ **Bucket Key Optimization** — Optional S3 Bucket Key to reduce KMS encryption costs
- ✅ **Access Logging** — Optional S3 access logs stored in dedicated logging bucket

### Optional Features

- ✅ **Static Website Hosting** — Enable with one parameter (configurable index document)
- ✅ **CI/CD Support** — Optional CI suffix for unique ephemeral deployments
- ✅ **Access Logging** — Optional server access logging to separate bucket with public access blocking

## Parameters

| Parameter | Type | Default | Required | Description |
| ----------- | ------ | --------- | ---------- | ------------- |
| `ProjectName` | String | — | ✅ Yes | Project name to use as bucket prefix |
| `BucketBaseName` | String | `cfn-bucket` | No | Base name for S3 bucket (1-20 chars, alphanumeric/dash/dot only) |
| `KmsKey` | String | `""` | No | KMS key for encryption: key name (`my-key`), alias (`alias/my-key`), or ARN (`arn:aws:kms:...`). Empty = SSE-S3 |
| `Environment` | String | `devl` | No | Deployment environment (devl, stag, prod, or custom) |
| `CiSuffix` | String | `""` | No | Optional CI suffix to append to bucket name for ephemeral deployments |
| `WebsiteConfiguration` | String | `false` | No | Enable static website hosting (`true` or `false`). Sets index document to `index.html` |
| `EnableLogging` | String | `false` | No | Enable S3 access logging (`true` or `false`). Creates dedicated logging bucket |
| `EnableBucketKey` | String | `false` | No | Enable S3 Bucket Key for KMS cost optimization (`true` or `false`). Only effective with KMS encryption |

**Notes:**
- Only `ProjectName` is required; all other parameters have sensible defaults
- `KmsKey` accepts three formats for maximum flexibility (auto-converts plain names to aliases)
- `EnableBucketKey` requires a valid `KmsKey` to have effect (reduces KMS API calls and costs)
- When `EnableLogging=true`, a separate logging bucket is created with identical security controls

## Outputs

| Output | Description |
| ----------- | ------------- |
| `S3BucketName` | Name of the created S3 bucket (exported for cross-stack references) |
| `S3BucketArn` | ARN of the created S3 bucket (exported for cross-stack references) |

## Usage

### 1. Upload Template to S3

```bash
aws s3 cp cloudformation/template.yaml s3://your-cfn-bucket/templates/s3-bucket.yaml
```

### 2. Reference from Parent Stack

In your parent/root CloudFormation template:

```yaml
S3BucketNestedStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/s3-bucket.yaml
    Parameters:
      ProjectName: !Ref ProjectName
      BucketBaseName: cfn-bucket
      Environment: !Ref Environment
      CiSuffix: !Ref CiSuffix
      KmsKey: !Ref KmsKeyParameter  # Optional: key name, alias, or ARN
      WebsiteConfiguration: "false"  # Optional: enable static hosting
      EnableLogging: "false"          # Optional: enable access logging
      EnableBucketKey: "false"        # Optional: reduce KMS costs
    Tags:
      - Key: Environment
        Value: !Ref Environment

Outputs:
  BucketName:
    Value: !GetAtt S3BucketNestedStack.Outputs.S3BucketName
    Export:
      Name: !Sub "${AWS::StackName}-BucketName"
  BucketArn:
    Value: !GetAtt S3BucketNestedStack.Outputs.S3BucketArn
    Export:
      Name: !Sub "${AWS::StackName}-BucketArn"
```

### 3. Deploy Using AWS CLI

#### Option A: Minimal Deployment (Default Security)

```bash
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-dev \
  --template-body file://cloudformation/template.yaml \
  --parameters file://cloudformation/parameters.json
```

#### Option B: Enable KMS Encryption

```bash
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-dev \
  --template-body file://cloudformation/template.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=myproject \
    ParameterKey=KmsKey,ParameterValue=alias/my-encryption-key \
    ParameterKey=EnableBucketKey,ParameterValue=true
```

#### Option C: Enable Access Logging

```bash
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-dev \
  --template-body file://cloudformation/template.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=myproject \
    ParameterKey=EnableLogging,ParameterValue=true
```

#### Option D: Enable Website Hosting

```bash
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-dev \
  --template-body file://cloudformation/template.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=myproject \
    ParameterKey=WebsiteConfiguration,ParameterValue=true
```

#### Option E: Full-Featured Deployment (All Options)

```bash
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-prod \
  --template-body file://cloudformation/template.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=myproject \
    ParameterKey=BucketBaseName,ParameterValue=cfn-bucket \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=KmsKey,ParameterValue=arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012 \
    ParameterKey=EnableBucketKey,ParameterValue=true \
    ParameterKey=EnableLogging,ParameterValue=true \
    ParameterKey=WebsiteConfiguration,ParameterValue=true
```

#### Option F: Ephemeral Deployment with CI Suffix

```bash
# For CI/CD pipelines (e.g., GitLab CI)
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-dev-ci-$CI_PIPELINE_ID \
  --template-body file://cloudformation/template.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=myproject \
    ParameterKey=CiSuffix,ParameterValue=$CI_PIPELINE_ID
```

## Bucket Naming Convention

The templates generate bucket names using the following pattern:

**Without CI Suffix:**

```bash
{ProjectName}-{BucketBaseName}-{AccountId}-{Environment}-{Region}
```

Example: `myproject-cfn-bucket-123456789012-devl-us-east-1`

**With CI Suffix:**

```bash
{ProjectName}-{BucketBaseName}-{AccountId}-{Environment}-{Region}-{CiSuffix}
```

Example: `myproject-cfn-bucket-123456789012-devl-us-east-1-pipeline-12345`

## Best Practices Implemented

### Security

- ✅ **Versioning Always Enabled** — Protects against accidental deletion and enables recovery
- ✅ **Public Access Blocking** — All four options enabled by default to prevent exposure
- ✅ **Dual Encryption Options** — SSE-S3 (default, no cost) or SSE-KMS (customer-managed keys)
- ✅ **Flexible Key Management** — Accepts key names, aliases, or ARNs for maximum compatibility
- ✅ **Bucket Key Optimization** — Reduces KMS encryption costs with optional Bucket Key
- ✅ **Resource Tagging** — Automatic tags for cost allocation and resource management

### Operational

- ✅ **Smart Bucket Naming** — Deterministic naming with project, account, environment, and region
- ✅ **Access Logging** — Optional dedicated logging bucket with public access blocking
- ✅ **Optional Website Hosting** — Static website support when needed
- ✅ **CI/CD Support** — Optional CI suffix for ephemeral deployments
- ✅ **Cross-Stack References** — Exported outputs for nested stack integration
- ✅ **Comprehensive Metadata** — CFN Context documentation for architectural clarity

## License

MIT
