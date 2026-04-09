# China Express: Warp Reverse Cross-Border Tunnel

A routing solution that enables China-based WARP users to access mainland China resources with **full Zero Trust security enforcement**, via a dedicated CMI cross-border link — no split tunneling, no security blind spots.

## The Problem

| Issue | Impact |
|-------|--------|
| Cloudflare Gateway PoPs are outside mainland China | Domestic traffic hairpins through overseas nodes → **1,100 ms+ latency** |
| Split Tunnel (the obvious fix) | Bypasses Device Posture Checks and Gateway policies entirely |
| DNS-level proxying | Only handles HTTP/HTTPS — SSH, RDP, UDP all fail |

## The Solution

Warp Reverse uses L3/L4 native steering with `100.64.0.0/10` as a Magic IP layer, routing any TCP/UDP traffic through a Cloudflare Tunnel to CMI's HK connector, then through CMI's dedicated cross-border link to a Shanghai egress point.

```
WARP Client (China)
        │
        ▼  Zero Trust inspection active ✓
Cloudflare Gateway (HK)
        │  Hostname Route → CMI Tunnel (QUIC)
        ▼
CMI cloudflared Connector (HK)
        │  CMI dedicated cross-border link
        ▼
Shanghai Egress → Ali DNS → Mainland China Target
```

**Result:** ~132 ms to AWS China Console (vs 1,100 ms), full protocol support, Zero Trust policies remain fully enforced.

## Documents

| Document | Description |
|----------|-------------|
| [**Solution Overview**](docs/warp-reverse-solution.md) | Full architecture, traffic flow, configuration steps, Hostname Routes API, tunnel token security, and performance results |
| [Tunnel Token Security Guide (EN)](docs/tunnel-token-security-guide-en.md) | Step-by-step deployment: AWS Secrets Manager + Docker + CMI compliance commitments |
| [Tunnel Token 安全部署手册（中文）](docs/tunnel-token-security-guide-zh.md) | 中文完整操作手册 |
| [China Domain List](docs/china-domains.md) | 200+ curated mainland China root domains for Hostname Routes — bulk upload ready |

## Quick Start

1. Read the [Solution Overview](docs/warp-reverse-solution.md) for the full configuration guide
2. Work with CMI to establish the cloudflared connector and cross-border link
3. Add mainland China domains via [Hostname Routes API](docs/warp-reverse-solution.md#api-recommended-for-bulk-upload) using the [China Domain List](docs/china-domains.md)
4. Remove `100.64.0.0/10` from Split Tunnel exclusions

## Performance

| Scenario | Without | With |
|----------|---------|------|
| `console.amazonaws.cn` | ~1,100 ms | **~132 ms** |
| `chinatax.gov.cn:8443` | Inaccessible | **Fully accessible** |
| Protocol support | HTTP/HTTPS only | **TCP / UDP / SSH / RDP** |
| Zero Trust enforcement | Bypassed (split tunnel) | **Fully active** |

---

*Cloudflare / CMI joint solution · Internal reference*
