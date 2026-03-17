# caddy-aws

[![Build and Push](https://github.com/hayward-solutions/caddy-aws/actions/workflows/build-and-push.yml/badge.svg)](https://github.com/hayward-solutions/caddy-aws/actions/workflows/build-and-push.yml)
[![Test](https://github.com/hayward-solutions/caddy-aws/actions/workflows/test.yml/badge.svg)](https://github.com/hayward-solutions/caddy-aws/actions/workflows/test.yml)

Custom [Caddy](https://caddyserver.com/) Docker image with AWS plugins for automatic TLS certificate management.

- **DNS-01 challenge** via [Route53](https://github.com/caddy-dns/route53) — no need to expose port 80
- **Certificate storage** in [S3](https://github.com/ss098/certmagic-s3) — persistent across container restarts and shared across instances
- **Multi-platform** — supports `linux/amd64` and `linux/arm64`

## Quick Start

```bash
docker run -d \
  -p 443:443 \
  -e CADDY_DOMAIN=example.com \
  -e CADDY_EMAIL=admin@example.com \
  -e CADDY_UPSTREAM=app:8080 \
  -e S3_BUCKET=my-caddy-certs \
  ghcr.io/hayward-solutions/caddy-aws:latest
```

> When running on EC2/ECS with an IAM role attached, AWS credentials are picked up automatically. See [IAM Roles](#iam-roles) below.

## Build from Source

```bash
docker build -t caddy-aws .
```

## Configuration

All configuration is done via environment variables. No Caddyfile editing required.

### Required

| Variable | Description |
|---|---|
| `CADDY_DOMAIN` | Domain to serve (e.g., `example.com` or `*.example.com`) |
| `CADDY_EMAIL` | Email for ACME account (Let's Encrypt) |
| `CADDY_UPSTREAM` | Reverse proxy upstream (e.g., `app:8080`) |
| `AWS_ACCESS_KEY_ID` | AWS access key (not needed with IAM roles) |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key (not needed with IAM roles) |
| `S3_BUCKET` | S3 bucket name for certificate storage |

### Optional

| Variable | Default | Description |
|---|---|---|
| `AWS_REGION` | `us-east-1` | AWS region for Route53 and S3 |
| `S3_HOST` | `s3.amazonaws.com` | S3 endpoint (change for S3-compatible stores) |
| `S3_PREFIX` | `caddy` | Key prefix for stored certificates |
| `S3_USE_IAM` | `true` | Set to `false` to use explicit credentials instead of IAM roles for S3 |

## Usage

### Docker Run

```bash
docker run -d \
  -p 443:443 \
  -e CADDY_DOMAIN=example.com \
  -e CADDY_EMAIL=admin@example.com \
  -e CADDY_UPSTREAM=app:8080 \
  -e AWS_ACCESS_KEY_ID=AKIA... \
  -e AWS_SECRET_ACCESS_KEY=... \
  -e AWS_REGION=us-east-1 \
  -e S3_BUCKET=my-caddy-certs \
  ghcr.io/hayward-solutions/caddy-aws:latest
```

### Docker Compose

```yaml
services:
  caddy:
    image: ghcr.io/hayward-solutions/caddy-aws:latest
    ports:
      - "443:443"
    environment:
      CADDY_DOMAIN: example.com
      CADDY_EMAIL: admin@example.com
      CADDY_UPSTREAM: app:8080
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
      AWS_REGION: us-east-1
      S3_BUCKET: my-caddy-certs

  app:
    image: my-app
    expose:
      - "8080"
```

## Custom Caddyfile

The default Caddyfile provides a single reverse proxy site. For more complex configurations, mount your own:

```bash
docker run -v ./Caddyfile:/etc/caddy/Caddyfile ghcr.io/hayward-solutions/caddy-aws:latest
```

The `{$VAR}` and `{$VAR:default}` substitution syntax works in custom Caddyfiles too.

## IAM Roles

IAM roles are used by default. When running on EC2 or ECS with an IAM role attached, just omit `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`:

```bash
docker run -d \
  -p 443:443 \
  -e CADDY_DOMAIN=example.com \
  -e CADDY_EMAIL=admin@example.com \
  -e CADDY_UPSTREAM=app:8080 \
  -e S3_BUCKET=my-caddy-certs \
  ghcr.io/hayward-solutions/caddy-aws:latest
```

The Route53 plugin uses the AWS SDK credential chain and picks up IAM role credentials automatically. The S3 storage plugin uses its IAM provider by default. To use explicit credentials instead, set `S3_USE_IAM=false` and provide `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

## AWS IAM Permissions

### IAM Policy

The following IAM policy provides the minimum permissions required for Caddy to manage DNS challenges and store certificates. Replace `HOSTED_ZONE_ID` with your Route53 hosted zone ID and `my-caddy-certs` with your S3 bucket name.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Route53GetChange",
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Sid": "Route53ManageTXTRecords",
      "Effect": "Allow",
      "Action": [
        "route53:ListResourceRecordSets",
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/HOSTED_ZONE_ID"
    },
    {
      "Sid": "Route53ListZones",
      "Effect": "Allow",
      "Action": "route53:ListHostedZonesByName",
      "Resource": "*"
    },
    {
      "Sid": "S3CertStorage",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-caddy-certs",
        "arn:aws:s3:::my-caddy-certs/*"
      ]
    }
  ]
}
```

### S3 Bucket Policy

This policy restricts bucket access to the Caddy IAM role and account administrators. Non-admin IAM users and other services cannot access the bucket. Replace `ACCOUNT_ID`, `CADDY_ROLE_NAME`, and `my-caddy-certs` with your values.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonCaddyNonAdmin",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-caddy-certs",
        "arn:aws:s3:::my-caddy-certs/*"
      ],
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::ACCOUNT_ID:role/CADDY_ROLE_NAME",
            "arn:aws:iam::ACCOUNT_ID:root",
            "arn:aws:iam::ACCOUNT_ID:role/Admin*"
          ]
        }
      }
    }
  ]
}
```

Adjust the `Admin*` pattern to match your admin role naming convention (e.g., `AdministratorAccess`, `AdminRole`).
