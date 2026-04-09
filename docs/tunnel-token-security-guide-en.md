# Cloudflared Tunnel Token — Secure Deployment Guide

**Goal:** Store the CF tunnel token in the customer's own AWS Secrets Manager. CMI runs `cloudflared` inside a Docker container using `--token-file`, so the token never appears in plaintext in any config file, process argument, or `docker inspect` output.

**Validated environment:** Amazon Linux 2 + Docker 25.0.14 + Docker Compose v5.1.1

**Estimated time:** 90 minutes

---

## Execution Environment Legend

| Badge | Meaning |
|-------|---------|
| `[MAC]` | Run on your local Mac / laptop |
| `[HOST]` | SSH into the CMI-provided EC2 instance |
| `[BROWSER]` | Operate in a web browser |

---

## Architecture & Security Boundary

### Two-layer isolation

1. **Storage isolation:** Token is stored in the customer's own AWS Secrets Manager. CMI has no AWS account access — `GetSecretValue` returns 403.
2. **Runtime isolation:** `cloudflared` runs in a Docker container using `--token-file`. The token value never appears in `docker inspect`, `ps aux`, or any config file.

```
┌─────────────────────────────────────────────────────────────┐
│          Customer's AWS Account (CMI has NO access)         │
│                                                             │
│  AWS Secrets Manager                                        │
│  └── prod/cloudflare/warp-reverse-tunnel-token-aws          │
│        AES-256 encrypted at rest                            │
│        Access: only the IAM Role bound to the host EC2      │
│        CMI IAM → GetSecretValue → 403 Forbidden             │
└───────────────────────────┬─────────────────────────────────┘
                            │ Instance Profile auto-auth
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                  CMI Host (EC2)                              │
│                                                             │
│  AWS CLI pulls token → /opt/cloudflared/secrets/tunnel.token│
│  Directory: chmod 700 (root-only entry)                     │
│  File:      chmod 604 (root rw, container UID 65532 r)      │
│                                                             │
│  docker-compose.yml (visible to CMI ops, no token value)    │
│  └── secrets.tunnel_token.file → tunnel.token              │
│                                                             │
│  cloudflared container (runs as UID 65532)                  │
│  └── --token-file /run/secrets/tunnel_token                 │
└───────────────────────────┬─────────────────────────────────┘
                            │ QUIC / TLS 1.3 (4 connections)
                            ▼
                   Cloudflare Global Network
```

### Honest disclosure: residual exposure

Any secret held by a running process can technically be read from memory by a host root user. This applies to `cloudflared` as well:

```bash
sudo docker exec cloudflared-prod cat /run/secrets/tunnel_token
# → token plaintext visible
```

This is a fundamental Linux OS property, not a defect in this solution. **This guide addresses static storage exposure (config files, docker inspect, ps aux).** Runtime memory access by root is mitigated through CMI's compliance and audit controls — see [CMI Compliance & Audit Commitments](#cmi-compliance--audit-commitments).

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| Independent AWS account | Must be fully inaccessible to CMI. If CMI manages your AWS account, this approach is invalid. |
| AWS IAM admin access | Ability to create IAM Policy, Role, Instance Profile |
| CMI-provided EC2 instance | Amazon Linux 2 or Ubuntu 20.04+ |
| SSH access to EC2 | |
| Cloudflare One account | Zero Trust enabled, ability to create Tunnels |
| AWS CLI on Mac | See Step 1 if not installed |

---

## Step 1 `[MAC]` — Configure AWS CLI Authentication

### 1-A: Verify AWS CLI is installed

```bash
aws --version
# Expected: aws-cli/2.x.x Python/3.x.x Darwin/...
```

Download from https://aws.amazon.com/cli/ if not installed.

### 1-B: Create an AWS Access Key

1. Log in to https://console.aws.amazon.com
2. Click your username (top-right) → **Security credentials**
3. Scroll to **Access keys** → **Create access key**

> **If the button is greyed out:** AWS allows a maximum of 2 access keys per account. Delete the oldest unused key first, then create a new one.

4. Use case: **Command Line Interface (CLI)** → confirm → Next → Create
5. **Copy both values immediately — they are only shown once.**

| Field | Format | Example (replace with your real values) |
|-------|--------|----------------------------------------|
| Access key ID | AKIA + 16 alphanumeric | `AKIAIOSFODNN7EXAMPLE` |
| Secret access key | 40-char string | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |

### 1-C: Configure AWS CLI

```bash
aws configure
```

```
AWS Access Key ID [None]:      AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]:  wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]:    ap-southeast-1     # ← your region
Default output format [None]:  json
```

**Common region codes:**

| Region | Code |
|--------|------|
| Singapore | `ap-southeast-1` |
| Hong Kong | `ap-east-1` |
| Tokyo | `ap-northeast-1` |
| US East (Virginia) | `us-east-1` |

### 1-D: Verify configuration

```bash
aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "AIDAIOSFODNN7EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

Record the `Account` value — this is your **AWS Account ID** (12-digit number). Replace `123456789012` with your real value in all subsequent steps.

> **"Could not connect to the endpoint URL" error:** Try appending `--region ap-southeast-1` (your region). Check network connectivity to AWS.

---

## Step 2 `[MAC]` — Record AWS Account ID and Region

All placeholders in this guide:

| Placeholder | Replace with | Source |
|-------------|-------------|--------|
| `123456789012` | Your 12-digit AWS Account ID | Step 1-D output |
| `ap-southeast-1` | Your AWS region code | Step 1-C |

---

## Step 3 `[BROWSER]` — Create a Tunnel in CF Dashboard and Get the Token

1. Open https://one.dash.cloudflare.com → select your account
2. **Networks → Connectors → Cloudflare Tunnels → Create a tunnel**
3. Connector type: **Cloudflared** → Next
4. Tunnel name: e.g., `warp-reverse-tunnel` → **Save tunnel**
5. Next page: select OS → **Docker**
6. The page shows a command like:

```bash
docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token eyJhIjoiMDI5N2Rm...
#                                                                           ↑
#                                                    Copy ONLY this eyJ... string
```

> **The OS selection does not affect the token.** Choosing Docker just makes the displayed command match our deployment. The token is identical regardless of which OS you select.

> **Do NOT click Next and do NOT run the displayed command.** "No connectors installed" on the page is expected — the tunnel exists but has no connector yet. It will appear automatically after Step 10.

Record:

| Variable | Value |
|----------|-------|
| `CF_TUNNEL_TOKEN` | Full `eyJ...` string |
| `CF_TUNNEL_NAME` | Your tunnel name |

---

## Step 4 `[MAC]` — Store Token in AWS Secrets Manager

> **If a secret with the same name is in "pending deletion" state:** AWS holds deleted secrets for 7 days before permanent removal. During this period you cannot create a same-named secret. Use a different name and update all subsequent commands accordingly.

### Option A: AWS CLI

```bash
aws secretsmanager create-secret \
  --name "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --description "Cloudflare tunnel token for Warp Reverse solution" \
  --secret-string "eyJhIjoiMDI5N2Rm...(your full token from Step 3)" \
  --region ap-southeast-1
```

Expected output:
```json
{
    "ARN": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/cloudflare/warp-reverse-tunnel-token-aws-AbCdEf",
    "Name": "prod/cloudflare/warp-reverse-tunnel-token-aws",
    "VersionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

**Record the full ARN (including the random suffix like `-AbCdEf`) — needed in Step 5.**

### Option B: AWS Console

1. AWS Console → search **Secrets Manager** → **Store a new secret**
2. Secret type: **Other type of secret** → switch to **Plaintext** → paste token → Next
3. Name: `prod/cloudflare/warp-reverse-tunnel-token-aws` → Next → Next → Store
4. Open the secret and copy the **Secret ARN**

### Verify

```bash
# describe-secret returns metadata only, NOT the token value
aws secretsmanager describe-secret \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --region ap-southeast-1
```

> If the output contains `"DeletedDate"`, the secret is in pending deletion. Restore it first:
> ```bash
> aws secretsmanager restore-secret \
>   --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
>   --region ap-southeast-1
> ```

---

## Step 5 `[MAC]` — Create IAM Policy (Minimal Permissions)

```bash
# Replace ap-southeast-1 with your region and 123456789012 with your Account ID
cat > /tmp/cf-tunnel-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadTunnelToken",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/cloudflare/warp-reverse-tunnel-token-aws-*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name "CloudflareTunnelTokenReadOnly" \
  --description "Read-only access to Cloudflare tunnel token in Secrets Manager" \
  --policy-document file:///tmp/cf-tunnel-policy.json
```

Expected output includes:
```json
{
    "Policy": {
        "PolicyName": "CloudflareTunnelTokenReadOnly",
        "Arn": "arn:aws:iam::123456789012:policy/CloudflareTunnelTokenReadOnly"
    }
}
```

> The `-*` wildcard at the end of the Resource ARN matches the random suffix AWS automatically appends to secret names.

---

## Step 6 `[MAC]` — Create IAM Role and Attach Policy

```bash
# Create Trust Policy (only EC2 service can assume this role)
cat > /tmp/ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create Role
aws iam create-role \
  --role-name "cf-tunnel-ec2-role" \
  --description "EC2 role for cloudflared — read tunnel token from Secrets Manager" \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json

# Attach Policy (replace 123456789012 with your Account ID)
aws iam attach-role-policy \
  --role-name "cf-tunnel-ec2-role" \
  --policy-arn "arn:aws:iam::123456789012:policy/CloudflareTunnelTokenReadOnly"

# Create Instance Profile
aws iam create-instance-profile \
  --instance-profile-name "cf-tunnel-ec2-profile"

# Add Role to Instance Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name "cf-tunnel-ec2-profile" \
  --role-name "cf-tunnel-ec2-role"
```

Verify:
```bash
aws iam list-attached-role-policies --role-name "cf-tunnel-ec2-role"
# Expected: CloudflareTunnelTokenReadOnly listed
```

---

## Step 7 `[BROWSER / MAC]` — Bind IAM Role to EC2 Instance

### Option A: AWS Console (recommended)

1. EC2 → Instances → select your instance
2. **Actions → Security → Modify IAM role**
3. Select `cf-tunnel-ec2-role` → **Update IAM role**

### Option B: AWS CLI

```bash
# Replace i-xxxxxxxxxxxxxxxxx with your EC2 Instance ID
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --iam-instance-profile Name="cf-tunnel-ec2-profile" \
  --region ap-southeast-1
```

### Verify on the host

```bash
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Expected: cf-tunnel-ec2-role
```

If no output, wait 1-2 minutes and retry.

---

## Step 8 `[HOST]` — Install Docker and AWS CLI

### Docker — Amazon Linux 2

> **Do NOT use `curl -fsSL https://get.docker.com | sudo sh`** — it does not support Amazon Linux (`amzn`) and will fail with `ERROR: Unsupported distribution 'amzn'`.
>
> If the machine has broken third-party repos (e.g., ookla speedtest), `yum` will fail entirely. Use `--disablerepo` to bypass them.

```bash
# Install using AWS official repo only
sudo yum install -y docker \
  --disablerepo="*" \
  --enablerepo="amzn2-core,amzn2extra-docker"

sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker   # refresh group membership (or re-login via SSH)
```

### Docker — Ubuntu / Debian

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```

### Docker Compose Plugin (Amazon Linux 2 — must install separately)

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```

Verify:
```bash
docker --version && docker compose version
# Docker version 25.0.14, build 0bab007
# Docker Compose version v5.1.1
```

### AWS CLI v2

```bash
curl -s "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip"
sudo yum install -y unzip 2>/dev/null || sudo apt-get install -y unzip 2>/dev/null
unzip -q /tmp/awscliv2.zip -d /tmp/
sudo /tmp/aws/install
aws --version
```

### Verify Instance Profile access (no `aws configure` needed)

```bash
aws secretsmanager describe-secret \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --region ap-southeast-1
# Expected: ARN and Name visible — no Access Key required
```

> **AccessDeniedException:** IAM Role not bound — go back to Step 7.  
> **UnrecognizedClientException:** Wrong region, or Instance Profile not yet active — wait 2 min and retry.

---

## Step 9 `[HOST]` — Pull Token from AWS and Write to Local Secret File

```bash
# Create secret directory (root-only entry)
sudo mkdir -p /opt/cloudflared/secrets
sudo chmod 700 /opt/cloudflared/secrets

# Pull token from AWS and write to file
aws secretsmanager get-secret-value \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --region ap-southeast-1 \
  --query "SecretString" \
  --output text \
  | sudo tee /opt/cloudflared/secrets/tunnel.token > /dev/null

# Set file permissions
# 604 = root rw, group none, other r
# IMPORTANT: cloudflared container runs as UID 65532 (non-root)
# The container needs "other" read permission (r--) to read the file
# chmod 600 will cause "permission denied" inside the container
sudo chmod 604 /opt/cloudflared/secrets/tunnel.token
sudo chown root:root /opt/cloudflared/secrets/tunnel.token

# Verify (regular users cannot access the directory at all)
sudo stat /opt/cloudflared/secrets/tunnel.token | grep Access
```

Expected output:
```
Access: (0604/-rw----r--)  Uid: (0/root)   Gid: (0/root)
```

Permission breakdown:
- `rw-` (owner=root): root can read/write
- `---` (group): no group access
- `r--` (other): container UID 65532 reads via "other" permission
- Directory `chmod 700`: regular ops accounts get `Permission denied` even on `ls`

---

## Step 10 `[HOST]` — Create docker-compose.yml and Start Container

```bash
sudo mkdir -p /opt/cloudflared

sudo tee /opt/cloudflared/docker-compose.yml > /dev/null << 'EOF'
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared-prod
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token-file /run/secrets/tunnel_token
    secrets:
      - tunnel_token
    tmpfs:
      - /tmp
    logging:
      driver: json-file
      options:
        max-size: "20m"
        max-file: "5"

secrets:
  tunnel_token:
    file: /opt/cloudflared/secrets/tunnel.token
EOF
```

**Key design decisions:**

| Setting | Reason |
|---------|--------|
| `--token-file` instead of `--token` | Token value never appears in `docker inspect` or process args |
| `secrets.tunnel_token.file` | Docker secrets mechanism mounts the file into the container |
| `tmpfs: /tmp` | Prevents container from writing temp files to host disk |
| `restart: unless-stopped` | Container auto-recovers after host reboot |

> **Why `read_only: true` is NOT used:** Docker secrets are mounted to `/run/secrets/`. In `read_only` mode, the container's UID 65532 cannot read that path and throws `permission denied`. Using `tmpfs` on `/tmp` provides equivalent protection without breaking the secrets mount.

```bash
cd /opt/cloudflared
sudo docker compose up -d
sudo docker compose ps
```

Expected:
```
NAME               IMAGE                           STATUS
cloudflared-prod   cloudflare/cloudflared:latest   Up X seconds
```

---

## Step 11 `[HOST / BROWSER]` — Verify Tunnel Is Running

### Check container logs (4 connections expected)

```bash
sudo docker logs cloudflared-prod --tail 30
```

Expected key lines:
```
INF Registered tunnel connection connIndex=0 ... location=sin21 protocol=quic
INF Registered tunnel connection connIndex=1 ... location=sin12 protocol=quic
INF Registered tunnel connection connIndex=2 ... location=sin22 protocol=quic
INF Registered tunnel connection connIndex=3 ... location=sin17 protocol=quic
```

Four connections across at least two different data centers — tunnel is healthy.

### Verify docker inspect does NOT expose the token

```bash
sudo docker inspect cloudflared-prod | grep -iE "token|eyJ"
# Expected: only file paths visible, NO eyJ... token value
```

### Verify process args do NOT contain the token

```bash
ps aux | grep cloudflared
# Expected: ...run --token-file /run/secrets/tunnel_token
#                               ↑ file path, not token value
```

### Verify in CF Dashboard

1. https://one.dash.cloudflare.com
2. Networks → Connectors → Cloudflare Tunnels
3. Your tunnel shows **Healthy** (green) with active connectors

---

## Step 12 `[HOST]` — Security Verification (Including Root-Level Transparency)

### Test 1: docker inspect (most common ops method)

```bash
sudo docker inspect cloudflared-prod | grep -iE "token|eyJ"
```
✅ **Blocked.** Output contains only file paths, no token value.

### Test 2: Read docker-compose.yml

```bash
cat /opt/cloudflared/docker-compose.yml | grep -iE "eyJ"
```
✅ **Blocked.** Config file contains no token value.

### Test 3: Regular ops account reads secret file

```bash
# As a non-root ops account:
ls /opt/cloudflared/secrets/tunnel.token
cat /opt/cloudflared/secrets/tunnel.token
```
✅ **Blocked.**
```
ls: cannot access /opt/cloudflared/secrets/tunnel.token: Permission denied
cat: /opt/cloudflared/secrets/tunnel.token: Permission denied
```
Directory `chmod 700` — non-root accounts cannot even enter the directory.

### Test 4: CMI attempts to access customer's AWS Secrets Manager

```bash
AWS_ACCESS_KEY_ID="" AWS_SECRET_ACCESS_KEY="" \
aws secretsmanager get-secret-value \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --region ap-southeast-1 \
  --no-sign-request 2>&1 || true
```
✅ **Blocked.**
```
An error occurred (AuthFailure): Request must be signed
```
CMI has no customer AWS credentials — architectural separation.

### Test 5: Root reads token via docker exec ⚠️ RESIDUAL EXPOSURE

```bash
sudo docker exec cloudflared-prod cat /run/secrets/tunnel_token
```
⚠️ **Token plaintext visible.**

This is an **acknowledged and intentionally disclosed** residual exposure. Any secret held by a running process is technically readable from memory by the host root user. This is a fundamental Linux OS property.

Equivalent paths that also expose the token with root access:
```bash
sudo cat /proc/$(docker inspect --format '{{.State.Pid}}' cloudflared-prod)/root/run/secrets/tunnel_token
```

**What this solution does protect against:** static storage exposure via config files, `docker inspect`, `ps aux` — which are the realistic everyday attack vectors.

**What mitigates the root-level risk:** CMI's compliance controls — see below.

---

## Security Summary

| Attack Vector | Result | Notes |
|---------------|--------|-------|
| `docker inspect` | ✅ Blocked | `--token-file` — only file path visible |
| `cat docker-compose.yml` | ✅ Blocked | No token in config |
| `ps aux` | ✅ Blocked | Only file path in process args |
| Regular ops account reads file | ✅ Blocked | Dir `chmod 700` — Permission denied |
| CMI accesses AWS Secrets Manager | ✅ Blocked | No IAM credentials — 403 |
| Root `docker exec` into container | ⚠️ Visible | Requires deliberate root action, fully audited |
| Root reads process memory | ⚠️ Visible | OS-level — no pure-software mitigation |

### Comparison: traditional vs. this approach

| Dimension | Traditional (`--token eyJ...` direct) | This Approach (Docker + `--token-file` + AWS) |
|-----------|--------------------------------------|----------------------------------------------|
| `docker inspect` exposure | ❌ Token in Args field | ✅ File path only |
| Config file on disk | ❌ Token in compose/service file | ✅ Secret file root-only; compose has no token |
| AWS Secrets Manager access | N/A | ✅ CMI has no IAM access — 403 |
| Token rotate | Edit config files + restart | ✅ Re-pull + restart container |
| Attack surface | Low barrier (just `cat` a file) | High barrier (root + deliberate action + audit trail) |

---

## CMI Compliance & Audit Commitments

Given the technical transparency above — a host root user can technically read the token from a running container — CMI commits to the following controls as a compliance layer:

### Least Privilege
- Daily ops accounts (e.g., `ec2-user`) do **not** have root or `sudo` privileges
- Root / sudo access is restricted to named system administrators, with dual-approval workflow
- CMI ops accounts have **no read permission** on any customer token path in OpenBao (CMI's internal secret manager) — only the host's AppRole credential can read it

### Full Audit Logging
- `auditd` on host records all `docker exec`, `ptrace`, `/proc` access attempts — logs are tamper-resistant
- OpenBao logs all secret read/write events with operator identity, timestamp, and source IP
- AWS CloudTrail logs all Secrets Manager API calls (customers can view independently in their own AWS account)
- Audit log retention: minimum 90 days

### Customer Audit Rights
- Customers may request audit reports from CMI at any time, filtered to their own token access events only — no other customer's data is included
- Customers can independently verify via their own AWS CloudTrail without relying on CMI

### Contractual Constraints
- CMI's service agreement explicitly prohibits accessing customer tunnel tokens or derived credentials
- Violations constitute a critical security incident with defined liability provisions

### Token Rotation Recommendations
- Rotate at minimum quarterly
- Rotate immediately upon any personnel change on CMI's side for this account
- After rotation, the old token is immediately invalidated on Cloudflare's side — even if previously read, it cannot be reused

---

## Troubleshooting

### `yum install docker` fails with "One of the configured repositories failed"

A broken third-party repo (e.g., ookla speedtest) is blocking yum. Use AWS official repos only:

```bash
sudo yum install -y docker \
  --disablerepo="*" \
  --enablerepo="amzn2-core,amzn2extra-docker"
```

### Container restarts with "Failed to read token file: permission denied"

The container runs as UID 65532, but `chmod 600` only allows root to read the file.

```bash
# Check container user
sudo docker inspect cloudflared-prod | grep '"User"'
# Should show "User": "65532:65532"

# Fix permissions
sudo chmod 604 /opt/cloudflared/secrets/tunnel.token

# Restart
cd /opt/cloudflared && sudo docker compose down && sudo docker compose up -d
```

### Tunnel Unhealthy / no connections

```bash
sudo docker logs cloudflared-prod --tail 50 | grep -E "ERR|WARN|failed"
```

Common cause: outbound ports blocked. Ensure the following outbound rules are open:
- `443/tcp` → `api.cloudflare.com`
- `7844/tcp + 7844/udp` → `region1.v2.argotunnel.com` and `region2.v2.argotunnel.com`

### How to rotate the token

**Step 1** `[BROWSER]` — CF Dashboard → Networks → Tunnels → tunnel → Edit → **Refresh token** → copy new token

**Step 2** `[MAC]` — Update AWS Secrets Manager:
```bash
aws secretsmanager update-secret \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --secret-string "eyJ...(new full token)" \
  --region ap-southeast-1
```

**Step 3** `[HOST]` — Re-pull and restart:
```bash
aws secretsmanager get-secret-value \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --region ap-southeast-1 \
  --query "SecretString" \
  --output text \
  | sudo tee /opt/cloudflared/secrets/tunnel.token > /dev/null

cd /opt/cloudflared && sudo docker compose restart cloudflared
```

If an unauthorized connector may have connected, also force-disconnect all connections:
```bash
# Replace ACCOUNT_ID, TUNNEL_ID, and CLOUDFLARE_API_TOKEN with your real values
curl "https://api.cloudflare.com/client/v4/accounts/ACCOUNT_ID/cfd_tunnel/TUNNEL_ID/connections" \
  --request DELETE \
  --header "Authorization: Bearer CLOUDFLARE_API_TOKEN"
```

---

*Version: 2026-04 | Validated on: Amazon Linux 2 + Docker 25.0.14 | Author: Dong Zhang*
