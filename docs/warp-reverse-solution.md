# China Express: Warp Reverse — Solution Overview

## Executive Summary

Warp Reverse provides a protocol-agnostic, low-latency path for China-based WARP users to access mainland China resources — while keeping full Zero Trust security enforcement active. It leverages **Cloudflare Tunnel (`cloudflared`)** and **Hostname Routing** to route domestic Chinese traffic through a dedicated CMI cross-border link, rather than relying on split tunneling (which bypasses Gateway policies).

---

## Background & Core Challenges

Enterprise customers in mainland China face a performance-security dilemma with Cloudflare Zero Trust:

| Challenge | Detail |
|-----------|--------|
| **Routing detours** | Cloudflare Gateway PoPs are outside China. A Shanghai→Beijing request may hairpin through Europe or the US, causing latency exceeding **1,100 ms**. |
| **Split tunnel security gap** | Excluding domestic traffic via Split Tunnel bypasses Device Posture Checks and Gateway Network Policies entirely. |
| **Protocol limitations** | DNS-level proxying only handles HTTP/HTTPS. Enterprise stacks require SSH, RDP, UDP (e.g., Tencent Meeting), and proprietary sockets. |

---

## Technical Solution: L3/L4 Native Steering

Warp Reverse operates at the network layer. It uses the `100.64.0.0/10` (CGNAT) range as a "Magic IP" to encapsulate any TCP/UDP traffic into a private QUIC tunnel, bypassing proxy overhead entirely.

### Traffic Flow

```
① WARP client initiates request to domestic domain (e.g., taobao.com)
         │
② Cloudflare Gateway identifies domain as a Hostname Route
   → Returns a "Magic IP" from 100.64.0.0/10 instead of the real IP
         │
③ Client sends TCP/UDP to the Magic IP
   → Gateway routes it into the CMI Cloudflare Tunnel
         │
④ Traffic exits at CMI's HK connector (cloudflared)
   → Forwarded through CMI's dedicated cross-border link
         │
⑤ Shanghai egress point → Ali DNS resolution → Mainland China target
```

![Architecture Diagram](../assets/warp-reverse-architecture.png)

---

## Implementation & Configuration

### Step 1: Establish the Dedicated Tunnel (CMI Operation)

The customer creates a Cloudflare Tunnel in Zero Trust Dashboard and hands the token to CMI. CMI runs `cloudflared` on their HK connector node to establish the tunnel.

> **For token security details, see [Step 1-A: Tunnel Token Handling](#step-1a-tunnel-token-handling--shared-responsibility) below.**

Navigate to **Zero Trust → Networks → Tunnels**, create a new Connector (type: Cloudflared), copy the token, and hand it to CMI.

CMI can provision multiple `cloudflared` replicas across geographically diverse endpoints for active-active load sharing and failover redundancy.

---

### Step 1-A: Tunnel Token Handling & Shared Responsibility

The tunnel token is a sensitive credential. By design, **any process running the token — including a containerized `cloudflared` — holds it in memory at runtime**, which means host-level administrative access could theoretically expose it. We believe in being transparent about this.

#### What CMI does to protect the token

CMI operates a dedicated token ingestion pipeline that eliminates all common **static** exposure vectors:

```
Customer → HTTPS Upload Portal → OpenBao (encrypted vault) → Docker secret → cloudflared
                                       │
                                       └─ AppRole scoping: ops staff have NO read access
                                          All access events written to immutable audit log
```

- The token is submitted via an **IP-allowlisted HTTPS portal** and stored encrypted in **OpenBao**. CMI operations staff have no read permission on any customer token path.
- `cloudflared` runs in an isolated Docker container using **`--token-file`** — the token value never appears in `docker inspect`, process arguments, or any config file.
- Every secret access event is recorded in an **immutable audit log**. Customers may request their own access records at any time.

#### Honest disclosure: residual exposure

Any secret held by a running process can technically be read from memory by a host root user. This is a fundamental Linux OS property — not a defect in this solution or in cloudflared. Specifically:

```bash
sudo docker exec cloudflared-prod cat /run/secrets/tunnel_token
# → token plaintext visible to root
```

This guide addresses the far more prevalent risk of **static storage exposure**. The root-level runtime risk is mitigated through CMI's compliance controls below.

#### Shared Responsibility

| | Customer | CMI |
|--|----------|-----|
| **Responsibility** | Generate & submit token; rotate periodically or upon any suspected exposure ([Cloudflare guidance](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/remote-tunnel-permissions/)) | Store under strict access controls; maintain audit logs; isolate each customer's connector |

#### Option: Customer-Managed Storage (AWS Secrets Manager)

Customers requiring a stronger trust boundary may store their token in their own **AWS Secrets Manager** account. CMI retrieves it via an instance-bound IAM role at startup — CMI never holds the token at rest. CMI's runtime operations (Docker, `--token-file`, audit logging) are identical either way.

| | CMI OpenBao *(default)* | AWS Secrets Manager *(customer-managed)* |
|--|------------------------|------------------------------------------|
| Token stored in | CMI encrypted vault | Customer's own AWS account |
| CMI access to stored token | Denied by policy; audited | **Architecturally impossible** |
| Customer effort | Submit via portal | AWS account + IAM setup required |

> **Full step-by-step guide:** [Tunnel Token Security Deployment Guide (EN)](tunnel-token-security-guide-en.md) · [中文版](tunnel-token-security-guide-zh.md)

#### CMI Compliance & Audit Commitments

- **Least privilege:** Daily ops accounts have no root/sudo access. Root access requires named sysadmin + dual-approval.
- **Audit logging:** `auditd` on host records all `docker exec`, `ptrace`, `/proc` access. OpenBao logs all secret read events. AWS CloudTrail available for customer-managed storage.
- **Customer audit rights:** Customers may request filtered audit reports (their data only) at any time, and can verify independently via AWS CloudTrail.
- **Contractual constraint:** CMI's service agreement explicitly prohibits accessing customer tunnel tokens. Violations constitute a critical security incident with defined liability.
- **Token rotation:** CMI recommends quarterly rotation and immediate rotation upon any personnel change.

---

### Step 2: Configure Hostname Routes

Navigate to **Cloudflare One → Networks → Routes → Hostname Routes**.

Add the mainland China root domains required for acceleration. The routing engine supports **SNI-based matching at the root level** — adding `abc.com` automatically steers all subdomains through the tunnel.

#### Dashboard

Add domains one by one, or use the API for bulk operations (recommended for large lists).

#### API (Recommended for bulk upload)

The API is significantly faster than the Dashboard for adding many domains. Each call adds one route:

```bash
curl -s -X POST \
  "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/teamnet/routes/network" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "network": "virtual_network_id_here",
    "comment": "Warp Reverse - taobao.com",
    "type": "hostname",
    "hostname": "taobao.com",
    "tunnel_id": "'"${TUNNEL_ID}"'"
  }'
```

For bulk upload, loop over a domain list:

```bash
#!/bin/bash
# Required: ACCOUNT_ID, API_TOKEN, TUNNEL_ID, VNET_ID

DOMAINS=(
  "taobao.com"
  "jd.com"
  "qq.com"
  "weixin.qq.com"
  # ... add more
)

for domain in "${DOMAINS[@]}"; do
  curl -s -X POST \
    "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/teamnet/routes/network" \
    -H "Authorization: Bearer ${API_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{
      \"network\": \"${VNET_ID}\",
      \"comment\": \"Warp Reverse - ${domain}\",
      \"type\": \"hostname\",
      \"hostname\": \"${domain}\",
      \"tunnel_id\": \"${TUNNEL_ID}\"
    }" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('result',{}).get('network',''), d.get('success'))"
  sleep 0.2  # rate limit courtesy delay
done
```

> **API reference:** [Cloudflare API — Create Hostname Route](https://developers.cloudflare.com/api/resources/zero_trust/subresources/networks/subresources/hostname_routes/methods/create/)

> **Full China domain list:** See [china-domains.md](china-domains.md) for a curated list of mainland China root domains covering major platforms, e-commerce, cloud services, and government portals — ready to paste into the script above.

---

### Step 3: Modify Split Tunnel Settings

In **Split Tunnels** (Exclude mode), **remove** the IP range `100.64.0.0/10` from the bypass list.

**Why:** By default, device profiles include all RFC 1918 and local prefixes. The `100.64.0.0/10` range is where Hostname Routes resolve via the Magic IP mechanism. Removing it forces this range into the WARP tunnel for Gateway inspection and steering.

---

## Performance Results

| Scenario | Without Warp Reverse | With Warp Reverse |
|----------|---------------------|-------------------|
| AWS China Console (`console.amazonaws.cn`) | ~1,100 ms | **~132 ms** |
| Government Portal (`chinatax.gov.cn:8443`) | Inaccessible (geo-blocked) | **Fully accessible** |
| Protocol support | HTTP/HTTPS only (via split tunnel) | **Full TCP/UDP/SSH/RDP** |
| Security posture | Device posture bypassed | **Full Zero Trust enforcement** |

---

## Technical Considerations & Billing

- **Service scope:** This solution relies on dedicated cross-border bandwidth provided by CMI. It is a premium add-on distinct from standard WARP bandwidth.
- **Root domain logic:** Adding `abc.com` covers all subdomains. Prioritize high-traffic collaboration and infrastructure domains.
- **Magic WAN compatibility:** Warp Reverse logic is natively supported for Magic WAN sites. MW-sourced traffic can trigger the reverse tunnel identically to WARP client traffic.
- **Redundancy:** CMI provisions two `cloudflared` replicas across two hosts (cf-host-01 / cf-host-02) for active-active failover.

---

## Related Documents

| Document | Purpose |
|----------|---------|
| [Tunnel Token Security Guide (EN)](tunnel-token-security-guide-en.md) | Full step-by-step deployment: AWS Secrets Manager + Docker |
| [Tunnel Token 安全部署手册（中文）](tunnel-token-security-guide-zh.md) | 中文完整操作手册 |
| [China Domain List](china-domains.md) | Curated mainland China root domain list for Hostname Routes |
