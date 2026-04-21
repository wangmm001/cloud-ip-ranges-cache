# cloud-ip-ranges-cache

Daily snapshots of AWS, GCP, Azure, and Cloudflare public IP CIDR publications — all four clouds' authoritative "my network belongs to service X in region Y" data, mirrored by [`wangmm001/feedcache`](https://github.com/wangmm001/feedcache).

## Why mirror it

These files are authoritative for "which CIDRs belong to AWS EC2 in `us-east-1`", "which are Cloudflare's edges", etc. Each cloud publishes at its own cadence (AWS updates multiple times per week; Azure every Monday; GCP & Cloudflare irregularly). Git history here is a reproducible timeline of how those allocations shift.

## Layout

```
data/
  YYYY-MM-DD/
    aws.json.gz              # https://ip-ranges.amazonaws.com/ip-ranges.json
    gcp.json.gz              # https://www.gstatic.com/ipranges/cloud.json
    azure.json.gz            # ServiceTags_Public_*.json (scraped from Microsoft download page)
    cloudflare-v4.txt.gz     # https://www.cloudflare.com/ips-v4
    cloudflare-v6.txt.gz     # https://www.cloudflare.com/ips-v6
  current/
    aws.json.gz
    gcp.json.gz
    azure.json.gz
    cloudflare-v4.txt.gz
    cloudflare-v6.txt.gz
```

All five files for a date are **atomic**: if any fetch fails, the day is skipped and nothing is committed. Files are the upstream payloads verbatim (AWS/GCP/Azure: full JSON; Cloudflare: newline-separated CIDRs).

## Consume

```bash
# latest AWS CIDRs
curl -L https://raw.githubusercontent.com/wangmm001/cloud-ip-ranges-cache/main/data/current/aws.json.gz | zcat | jq '.prefixes[] | select(.service=="EC2" and .region=="us-east-1") | .ip_prefix' | head

# extract Cloudflare edge v4
curl -L https://raw.githubusercontent.com/wangmm001/cloud-ip-ranges-cache/main/data/current/cloudflare-v4.txt.gz | zcat
```

## Update cadence vs cron

Cron runs daily (04:45 UTC) but upstream cadences vary:

| Cloud | Upstream cadence | Typical commits/week |
|---|---|---|
| AWS | multiple times/week, has `syncToken` | 3–5 |
| Azure | weekly (Monday release) | 1 |
| GCP | irregular | ~1 |
| Cloudflare (v4/v6) | rare | <1 |

Deterministic gzip + `git diff --cached --quiet` means unchanged clouds produce no commit that day.

## License

Scaffolding (README, cron yml): MIT.

The mirrored data is each cloud's own publication and carries their respective terms — see AWS [`/ip-ranges/index.html`](https://docs.aws.amazon.com/vpc/latest/userguide/aws-ip-ranges.html), [Microsoft Azure download page](https://www.microsoft.com/en-us/download/details.aspx?id=56519), [GCP docs](https://cloud.google.com/vpc/docs/subnets), [Cloudflare IPs](https://www.cloudflare.com/ips/).

## How it works

Daily GitHub Actions cron (04:45 UTC) calls `wangmm001/feedcache`'s reusable workflow, which fetches all five blobs to memory atomically (Azure via a two-hop: scrape the Microsoft confirmation page to find the weekly-rotating JSON filename, then fetch it).
