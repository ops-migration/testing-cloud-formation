# Nested CloudFormation Stacks — Dev Environment

## Architecture Overview

```
master-stack.yaml
├── vpc-stack.yaml         (VPC, Subnets, IGW, NAT GW, Route Tables)
├── s3-stack.yaml          (S3 Bucket with encryption, versioning, HTTPS policy)
├── alb-stack.yaml         (ALB, Security Group, Target Group, HTTP Listener)
└── ecs-stack.yaml         (ECS Cluster, EC2 ASG t3.small 1/1/1, Capacity Provider)
```

## Files

| File                | Description                                        |
|---------------------|----------------------------------------------------|
| `master-stack.yaml` | Parent stack — orchestrates all child stacks       |
| `vpc-stack.yaml`    | VPC: 2 public + 2 private subnets across 2 AZs    |
| `s3-stack.yaml`     | Encrypted, versioned S3 app bucket                 |
| `alb-stack.yaml`    | Internet-facing ALB with health-check target group |
| `ecs-stack.yaml`    | ECS EC2 cluster: t3.small, min=max=desired=1       |

---

## Pre-requisites

1. An existing S3 bucket to host the nested templates (referred to as `<TEMPLATES_BUCKET>`).
2. An EC2 Key Pair in your target region.
3. AWS CLI configured with sufficient permissions.

---

## Deployment Steps

### 1. Create a templates bucket (one-time)

```bash
aws s3 mb s3://<TEMPLATES_BUCKET> --region us-east-1
```

### 2. Upload nested templates

```bash
aws s3 cp vpc-stack.yaml  s3://<TEMPLATES_BUCKET>/templates/vpc-stack.yaml
aws s3 cp s3-stack.yaml   s3://<TEMPLATES_BUCKET>/templates/s3-stack.yaml
aws s3 cp alb-stack.yaml  s3://<TEMPLATES_BUCKET>/templates/alb-stack.yaml
aws s3 cp ecs-stack.yaml  s3://<TEMPLATES_BUCKET>/templates/ecs-stack.yaml
```

### 3. Deploy the master stack

```bash
aws cloudformation deploy \
  --template-file dev.yaml \
  --stack-name dev-environment \
  --parameter-overrides \
      Environment=dev \
      TemplatesBucketName=<TEMPLATES_BUCKET> \
      KeyPairName=<YOUR_KEY_PAIR> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

### 4. Verify outputs

```bash
aws cloudformation describe-stacks \
  --stack-name dev-environment \
  --query "Stacks[0].Outputs" \
  --output table
```

---

## Stack Dependency Graph

```
VPCStack ──────────────────────────────────────────┐
                                                   │
S3Stack (independent)                              │
                                                   ▼
ALBStack ─── depends on VPCStack ─────────────┐
                                              │
ECSStack ── depends on VPCStack + ALBStack ◄──┘
```

---

## Key Design Decisions

- **Single NAT Gateway**: Cost-saving measure appropriate for dev.
- **ECS EC2 on t3.small**: Cheaper than Fargate for always-on dev workloads; min/max/desired = 1.
- **Private subnets for ECS**: Container instances are not directly internet-accessible; traffic flows via ALB → Target Group → private instances.
- **S3 HTTPS-only policy**: Denies any insecure (HTTP) transport to the bucket.
- **Container Insights enabled**: Useful even in dev for debugging metrics.
- **Short CloudWatch log retention (7 days)**: Keeps dev costs low.

---

## Cleanup

```bash
aws cloudformation delete-stack --stack-name dev-environment
# Also delete the templates bucket if no longer needed:
aws s3 rb s3://<TEMPLATES_BUCKET> --force
```