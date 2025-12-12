# Infrastructure DNS

Multi-provider DNS management for alternatefutures.ai using OctoDNS.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    REGISTRAR (Namecheap)                     │
│              NS: Cloudflare (primary, active)                │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
        ┌──────────┐                    ┌──────────┐
        │Cloudflare│                    │  deSEC   │
        │  (active)│                    │(standby) │
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

If Cloudflare goes down, you can quickly switch to deSEC by changing NS at your registrar. Records are already synced and ready.

| Provider | Status | Role |
|----------|--------|------|
| Cloudflare | Active (NS at registrar) | Primary, serving traffic |
| deSEC | Standby (records synced) | Backup, ready to serve |

## Files

- `octodns.yaml` - OctoDNS configuration (Cloudflare + deSEC)
- `zones/alternatefutures.ai.yaml` - DNS records (source of truth)
- `.github/workflows/dns-sync.yml` - Syncs records to all providers on push
- `.github/workflows/dns-monitor.yml` - Monitors health every 5 minutes

## Usage

### Edit DNS Records

1. Edit `zones/alternatefutures.ai.yaml`
2. Push to main branch
3. GitHub Actions syncs to both Cloudflare and deSEC

### Local Development

```bash
# Install
pip install -r requirements.txt

# Validate
export CLOUDFLARE_API_TOKEN="..."
export DESEC_API_TOKEN="..."
octodns-validate --config=octodns.yaml

# Dry run
octodns-sync --config=octodns.yaml --doit=false

# Apply
octodns-sync --config=octodns.yaml --doit
```

## Required Secrets

Set these in GitHub repository secrets:

| Secret | Description |
|--------|-------------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token with Zone:DNS:Edit + Zone:Page Rules:Read |
| `DESEC_API_TOKEN` | deSEC API token |

## Provider Setup

### Cloudflare (Primary)
- Zone: `alternatefutures.ai` (active)
- NS: `jeremy.ns.cloudflare.com`, `miki.ns.cloudflare.com`
- Token requires: Zone:Zone:Read, Zone:DNS:Edit, Zone:Page Rules:Read

### deSEC (Secondary - Free)
- Zone: `alternatefutures.ai` (standby)
- NS: `ns1.desec.io`, `ns2.desec.org`
- Minimum TTL: 3600 seconds (1 hour)
- Docs: https://desec.readthedocs.io/

## Failover Procedure

If Cloudflare is down:

1. Log into Namecheap (registrar)
2. Go to Domain → Nameservers
3. Change NS to: `ns1.desec.io`, `ns2.desec.org`
4. Wait for propagation (5-60 minutes)

To switch back to Cloudflare:
1. Change NS to: `jeremy.ns.cloudflare.com`, `miki.ns.cloudflare.com`

## Monitoring

The `dns-monitor.yml` workflow runs every 5 minutes:
- Queries each provider's nameserver directly
- Creates GitHub issue with `dns-alert` label if providers are down
- Shows health status in workflow summary

## Notes

- TTL is set to 3600 seconds to satisfy deSEC's minimum requirement
- Zone file keys must be alphabetically sorted (`enforce_order: true`)
- OctoDNS loads all providers at startup, so both tokens must be set even when targeting one provider
