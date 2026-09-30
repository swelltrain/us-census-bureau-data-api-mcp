# Hosting Our Own Census MCP — Plan

## Overview

This document covers the architecture, sizing, authentication, and caching strategy for
self-hosting the `us-census-bureau-data-api-mcp` server on AWS EC2 for internal company use.

The MCP server is a Node.js 22 / TypeScript process that communicates over the MCP protocol,
backed by a PostgreSQL 16 database pre-loaded with Census Bureau metadata and a response cache.
Upstream data comes from `api.census.gov`.

---

## 1. EC2 Sizing

### Recommendation: `t3.large` (2 vCPU, 8 GB RAM)

| Scenario | Instance | Notes |
|---|---|---|
| Slim mode only (no ZCTA/Places/County Subdivisions) | `t3.small` (2 vCPU, 2 GB) | Tight; acceptable for a proof-of-concept |
| Full seed, light single-user use | `t3.medium` (2 vCPU, 4 GB) | Minimum viable production |
| **Full seed, multiple concurrent LLM sessions** | **`t3.large` (2 vCPU, 8 GB)** ← recommended | Postgres + pg_trgm GIN indexes are memory-hungry; trigram search is CPU-bound |

### Storage: 500 GB gp3 EBS

| Layer | Estimated Size |
|---|---|
| OS + Docker images | ~10 GB |
| PostgreSQL full seed (geographies + indexes) | ~600 MB |
| Cache — metadata layer (datasets, geographies) | ~500 MB |
| Cache — state/county ACS 5-year, last 3 years (pre-warmed) | ~5–20 GB |
| Cache — on-demand tract-level growth headroom | ~400+ GB |

gp3 gives 3,000 IOPS baseline at no extra cost and can be resized online if the cache grows.

### Why these numbers

- The `geographies` table holds ~100K+ rows (places, ZCTAs, county subdivisions) with pg_trgm
  GIN indexes — these are loaded into shared buffers and are memory-sensitive.
- The `pg` connection pool is configured at `max: 10` — Postgres needs headroom for all connections.
- The DB seeding process (one-time, at init) makes ~50 sequential per-state API calls for county
  subdivisions — this is the most CPU-intensive phase, but it is a one-time cost.
- `SEED_MODE=slim` can be used to skip ZCTAs, Places, and County Subdivisions if a smaller
  instance is preferred. This reduces the geography table to a fraction of its full size.

---

## 2. Architecture: HTTPS + API Key

Because the MCP server currently uses **stdio transport** (not HTTP), clients pipe stdin/stdout
directly into the container. To expose this securely over a network with API key authentication,
the transport must be changed to **HTTP/SSE** (Server-Sent Events), which the MCP SDK supports
natively alongside stdio.

### Target Architecture

```
                         ┌─────────────────────────────────────────┐
                         │                EC2 Instance              │
                         │                                          │
Client (Claude Desktop,  │   ┌──────────┐     ┌─────────────────┐  │
  Cursor, etc.)          │   │          │     │                 │  │
    │                    │   │  nginx   │────▶│   mcp-server    │  │
    │  HTTPS + API key   │   │  (443)   │     │  (HTTP/SSE)     │  │
    └───────────────────▶│   │          │     │   Node.js 22    │  │
                         │   └──────────┘     └────────┬────────┘  │
                         │        │                    │           │
                         │   API key validation        │           │
                         │   (Authorization header)    ▼           │
                         │                    ┌─────────────────┐  │
                         │                    │  PostgreSQL 16  │  │
                         │                    │  (cache + meta) │  │
                         │                    └────────┬────────┘  │
                         └─────────────────────────────┼───────────┘
                                                        │
                                                        ▼
                                               api.census.gov
                                           (upstream, rate-limited)
```

### What needs to change in the codebase

1. **Swap `StdioServerTransport` for `SSEServerTransport`** in `mcp-server/src/index.ts`.
   The `@modelcontextprotocol/sdk` already ships this transport — it is a small code change.

2. **Expose a port** in `docker-compose.yml` for the mcp-server container (e.g. `3000:3000`).

3. **Add nginx** as a reverse proxy container in the compose stack with:
   - TLS termination (cert via Let's Encrypt / Certbot, or ACM if behind an ALB)
   - API key validation via the `Authorization: Bearer <token>` header
   - A static map of valid keys (or a simple lookup against a keys table in Postgres)

### nginx API key validation (example)

```nginx
server {
    listen 443 ssl;
    server_name census-mcp.yourcompany.internal;

    ssl_certificate     /etc/letsencrypt/live/census-mcp.yourcompany.internal/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/census-mcp.yourcompany.internal/privkey.pem;

    location / {
        # Validate the bearer token
        if ($http_authorization != "Bearer YOUR_INTERNAL_API_KEY") {
            return 401 '{"error": "Unauthorized"}';
        }

        proxy_pass         http://mcp-server:3000;
        proxy_http_version 1.1;

        # Required for SSE (Server-Sent Events)
        proxy_set_header   Connection '';
        proxy_buffering    off;
        proxy_cache        off;
        chunked_transfer_encoding on;
    }
}
```

For multiple team members with individual keys, replace the static string check with a
`map` block or a Lua/auth_request sub-request against a simple keys table.

### AWS-level access controls

Even with API key auth, restrict exposure at the network level:

- Place the EC2 in a **private subnet** inside your company VPC
- Allow inbound 443 only from your office IP ranges or VPN CIDR via **Security Group rules**
- Optionally put an **Application Load Balancer** in front for TLS offloading and access logs

### Client configuration (Claude Desktop example)

```json
{
  "mcpServers": {
    "census": {
      "url": "https://census-mcp.yourcompany.internal/sse",
      "headers": {
        "Authorization": "Bearer YOUR_INTERNAL_API_KEY"
      }
    }
  }
}
```

---

## 3. Caching Strategy

### Current state

The `census_data_cache` table, its indexes, and stored functions (`generate_cache_hash`,
`cleanup_expired_cache`, `get_cache_stats`) are **fully defined in the database schema** but
**completely unwired in the application layer**. Every tool call hits `api.census.gov`
unconditionally. The `expires_at` column has no default, so `cleanup_expired_cache()` is
currently a no-op.

This is the single highest-leverage improvement available — wiring the cache in would
immediately eliminate redundant Census API calls and effectively remove the 500 calls/day
rate limit as a concern for repeated queries.

### Can we cache ALL Census data?

**Not entirely — the combinatorial space is too large.**

```
~40,000 tables × ~30 geography levels × ~30 years × many variable combos
= tens of millions of possible API calls
```

Tract-level responses for a single ACS table are 1–10 MB of JSON. Full coverage would
require low-TB storage and a months-long pre-warm run.

**However**, Census data is almost entirely static. The 2022 ACS will never change.
This makes a targeted pre-warm of the data your team actually uses extremely practical —
you are building a local replica of a data archive, not caching a live feed.

### What can realistically be cached

| Layer | Strategy | Estimated Size | Refresh Cadence |
|---|---|---|---|
| `list-datasets` | Cache permanently; refresh on new ACS vintage | ~5 MB | Yearly |
| `fetch-dataset-geography` | Cache per dataset/year combo | ~200–500 MB | Yearly |
| `fetch-aggregate-data` — state/county level | Pre-warm popular tables (ACS 5-yr, last 3 years) | ~5–20 GB | Yearly |
| `fetch-aggregate-data` — tract level | On-demand cache only; never pre-warm | Unbounded | Yearly |
| `resolve-geography-fips` / `search-data-tables` | Already DB-only — no upstream calls | — | — |

State + county level, ACS 5-year, last 3 years covers approximately **90% of real-world queries**
for most business and policy use cases.

### TTL strategy

| Data type | `expires_at` value | Rationale |
|---|---|---|
| Historical vintages (any year < current) | `NULL` (never expire) | Data is permanently frozen |
| Current-year ACS estimates | `NOW() + INTERVAL '1 year'` | Refresh after next vintage release |
| `list-datasets` catalog | `NOW() + INTERVAL '90 days'` | Census occasionally adds datasets |
| `fetch-dataset-geography` | `NULL` for closed years; 90 days for current | Same rationale |

### Phased implementation plan

#### Phase 1 — Wire the existing cache skeleton (small change, immediate win)

- Add cache read/write logic to `BaseTool` or each tool's `toolHandler` in `mcp-server/src/`
- Use the existing `generate_cache_hash()` DB function (or replicate its SHA-256 logic in TS)
  to produce a lookup key before every Census API call
- On cache hit: return stored `response_data` directly
- On cache miss: fetch from Census API, write to `census_data_cache` with appropriate `expires_at`
- Wire `cleanup_expired_cache()` as a scheduled job (cron inside the container, or a Postgres
  `pg_cron` extension job)

#### Phase 2 — Fix TTL population and add cache stats logging

- Set `expires_at` intelligently on every cache write (see TTL strategy above)
- Expose `get_cache_stats()` as a health/metrics endpoint or log it on startup
- Add a cache hit/miss counter to `api_call_log` for observability

#### Phase 3 — Pre-warm the high-value slice

Build a pre-warm script (`scripts/prewarm-cache.ts`) that:

1. Reads all seeded datasets from the `datasets` table
2. Filters to ACS 5-year vintages for the last 3 years
3. For each vintage, fetches state-level and county-level responses for the top ~200 tables
   (by query frequency or by a curated list)
4. Writes results directly to `census_data_cache` with `expires_at = NULL` for closed years
5. Runs once after initial deployment; re-runs each October when the new ACS vintage drops

This eliminates the Census API rate limit as a concern for the vast majority of queries.

---

## 4. Census API Key

The current architecture passes a single `CENSUS_API_KEY` environment variable shared across
all MCP clients. The Census Bureau's free tier allows **500 calls/day per key**.

Once Phase 1 caching is in place, repeated queries will be served from Postgres and will not
consume API quota. Pre-warming (Phase 3) will be a one-time burst — request a higher rate
limit from the Census Bureau (they grant increases for registered applications) or run the
pre-warm over several days.

Longer term, if the cache is comprehensive enough, the Census API key becomes a fallback
dependency only (for new/unusual queries) rather than a hot path.

---

## 5. Summary: Recommended Stack

| Component | Choice |
|---|---|
| EC2 instance | `t3.large` (2 vCPU, 8 GB RAM) |
| EBS volume | 500 GB gp3 |
| OS | Amazon Linux 2023 |
| Container runtime | Docker Compose (existing) |
| MCP transport | HTTP/SSE (swap from stdio) |
| Reverse proxy | nginx (TLS termination + API key validation) |
| TLS certificates | Let's Encrypt (Certbot) or ACM via ALB |
| Network access control | Security Group — inbound 443 from VPN CIDR only |
| Authentication | Static bearer token per team/user, validated in nginx |
| Caching | Phase 1 → Phase 3 as above; target 90%+ cache hit rate |
| Cache refresh | Yearly (new ACS vintage), automated via cron |
