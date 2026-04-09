# China Express: Warp Reverse Cross-Border Tunnel

A routing solution that enables China-based WARP users to access mainland China resources with full Zero Trust security enforcement, via a dedicated CMI cross-border link.

## Overview

- **Problem**: Cloudflare Gateway PoPs are outside mainland China. Domestic traffic suffers hairpinning (1,100 ms+), and Split Tunneling disables device posture checks and Gateway policies.
- **Solution**: L3/L4 native steering using Cloudflare Tunnel + Hostname Routes + CMI's dedicated cross-border link — full protocol support (TCP, UDP, SSH, RDP) with sub-150 ms latency.

## Architecture

```
WARP Client (China)
        │
        ▼
Cloudflare Gateway (HK/JP)
        │  Hostname Route → CMI Tunnel
        ▼
CMI cloudflared Connector (HK Node)
        │  CMI Cross-Border Link
        ▼
Shanghai Egress → Ali DNS → Mainland China Target
```

## Documents

| Document | Description |
|----------|-------------|
| [Warp Reverse — Solution Overview](docs/warp-reverse-solution.md) | Full architecture, traffic flow, configuration steps, and performance results |
| [Tunnel Token Security Deployment Guide (EN)](docs/tunnel-token-security-guide-en.md) | Step-by-step guide: secure token storage with AWS Secrets Manager + Docker deployment |
| [Tunnel Token 安全部署手册（中文）](docs/tunnel-token-security-guide-zh.md) | Chinese version of the deployment guide |

## Quick Start

See [Warp Reverse — Solution Overview](docs/warp-reverse-solution.md) for the complete setup guide.

For tunnel token security (CMI operators), see the [Token Security Guide](docs/tunnel-token-security-guide-en.md).

## Performance

| Scenario | Without Warp Reverse | With Warp Reverse |
|----------|---------------------|-------------------|
| AWS China Console (`console.amazonaws.cn`) | ~1,100 ms | ~132 ms |
| Government Portal (`chinatax.gov.cn:8443`) | Inaccessible (geo-blocked) | Fully accessible |

## License

Internal reference document — Cloudflare / CMI joint solution.
