# Infrastructure DNS

Multi-provider DNS management for alternatefutures.ai with automatic failover monitoring.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    REGISTRAR (OpenProvider)                  │
│        NS Records: cloudflare + ns1 (all active)            │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
        ┌──────────┐                    ┌──────────┐
        │Cloudflare│                    │   NS1    │
        │   Zone   │                    │   Zone   │
        └──────────┘                    └──────────┘
              │                               │
              └───────────────┬───────────────┘
                              │
                    ┌─────────┴─────────┐
                    │     OctoDNS       │
                    │  (record sync)    │
                    └───────────────────┘
```

## Why Multi-Provider DNS?

DNS outages happen. By using multiple providers with NS records from all of them, DNS resolvers automatically try alternates if one fails.

| Outage Scenario | Impact |
|-----------------|--------|
| Cloudflare down | Site works (NS1 serves) |
| NS1 down | Site works (Cloudflare serves) |
| Both down | Site down (extremely unlikely) |

## Files

- `octodns.yaml` - OctoDNS configuration
- `zones/alternatefutures.ai.yaml` - DNS records (source of truth)
- `.github/workflows/dns-sync.yml` - Syncs records to all providers
- `.github/workflows/dns-monitor.yml` - Monitors health every 5 minutes

## Usage

### Edit DNS Records

1. Edit `zones/alternatefutures.ai.yaml`
2. Push to main branch
3. GitHub Actions syncs to all providers

### Local Development

```bash
# Install
pip install -r requirements.txt

# Validate
octodns-validate --config=octodns.yaml

# Dry run
export CLOUDFLARE_API_TOKEN="..."
export NS1_API_KEY="..."
octodns-sync --config=octodns.yaml --doit=false

# Apply
octodns-sync --config=octodns.yaml --doit
```

## Required Secrets

Set these in GitHub repository secrets:

| Secret | Description |
|--------|-------------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token with DNS edit |
| `NS1_API_KEY` | NS1 API key |

## Provider Setup

### Cloudflare
1. Already configured for alternatefutures.ai
2. NS: `jeremy.ns.cloudflare.com`, `paisley.ns.cloudflare.com`

### NS1 (Free Tier - 500k queries/mo)
1. Sign up at https://ns1.com/signup
2. Add zone `alternatefutures.ai`
3. Create API key: Account Settings → API Keys → Add Key
4. NS: `dns1.p01.nsone.net`, `dns2.p01.nsone.net`, `dns3.p01.nsone.net`, `dns4.p01.nsone.net`

## Monitoring

The `dns-monitor.yml` workflow runs every 5 minutes:
- Queries each provider's nameserver directly
- Creates GitHub issue if providers are down
- Shows health status in workflow summary

## ACME Challenge Delegation

The `_acme-challenge.*` subdomains are delegated to Cloudflare via NS records.
This means Let's Encrypt DNS-01 challenges always go to Cloudflare, regardless
of which provider serves the main zone.

If Cloudflare is down:
- Site continues working (NS1 serves)
- Certificate renewal fails (existing certs valid 90 days)
- Once Cloudflare recovers, certs renew automatically
