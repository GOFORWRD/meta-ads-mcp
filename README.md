# Meta Ads MCP

A [Model Context Protocol](https://modelcontextprotocol.io/) server that exposes the Meta Marketing API
(Facebook / Instagram Ads) to MCP clients such as Claude Desktop, Claude Code, and Cursor — read campaign
performance, manage campaigns, ad sets and ads, upload creatives, and query targeting data through natural
language.

> ### Unofficial fork
>
> This is a modified fork of [pipeboard-co/meta-ads-mcp](https://github.com/pipeboard-co/meta-ads-mcp),
> maintained independently and **not supported by Pipeboard**. Please raise issues here rather than in
> Pipeboard's Discord or support channels.
>
> **Licence:** Business Source License 1.1 (see [LICENSE](LICENSE)), inherited from upstream. BSL is *not*
> an open-source licence — production use is permitted **except** offering the work to third parties on a
> hosted or embedded basis competing with the licensor's commercial offerings. It converts to Apache 2.0 on
> **1 January 2029**. Read the licence before deploying this commercially.
>
> Ad account IDs, page names and similar identifiers throughout the tests and docs are placeholders.

---

## How this fork differs from upstream

Upstream is built around Pipeboard's hosted, commercial MCP service — its README is largely about that
product. This fork drops all of it and documents **self-hosting only**:

| | Upstream | This fork |
|---|---|---|
| Primary path | Pipeboard hosted remote MCP | You run it yourself |
| Auth | Pipeboard account (`PIPEBOARD_API_TOKEN`) | Your own Meta app + access token |
| Deployment docs | Sign up for the service | Local stdio, or Docker + Caddy on a VPS |

The upstream Pipeboard code paths (`PIPEBOARD_API_TOKEN`, `meta_ads_mcp/core/pipeboard_auth.py`) still exist
in the source but are **not used or tested here**. You can ignore them entirely.

---

## Contents

- [Requirements](#requirements)
- [1. Create a Meta app and get a token](#1-create-a-meta-app-and-get-a-token)
- [2a. Run locally (stdio)](#2a-run-locally-stdio)
- [2b. Run on a VPS (Docker + Caddy)](#2b-run-on-a-vps-docker--caddy)
- [Connecting MCP clients](#connecting-mcp-clients)
- [Configuration reference](#configuration-reference)
- [Available tools](#available-tools)
- [Security notes](#security-notes)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)

---

## Requirements

- Python 3.11+ (local install) **or** Docker + Docker Compose (VPS)
- A Meta Developer app with Marketing API access
- A Meta access token with permission on the ad accounts you want to use

---

## 1. Create a Meta app and get a token

1. Go to [developers.facebook.com/apps](https://developers.facebook.com/apps) and create an app of type
   **Business**.
2. Add the **Marketing API** product.
3. From **Settings → Basic**, copy the **App ID** and **App Secret**.

### Getting an access token

The server accepts a Meta user access token. For ad reads and writes you generally need the
`ads_read` and `ads_management` permissions (plus `business_management` for account discovery).

The quickest route is the [Graph API Explorer](https://developers.facebook.com/tools/explorer/): select your
app, request those permissions, generate a token, then exchange it for a long-lived (≈60 day) token via
[Access Token Tool](https://developers.facebook.com/tools/accesstoken/) or the
`oauth/access_token` endpoint.

> **Note on the built-in `--login` flow.** This repo ships `python -m meta_ads_mcp --login`, which runs a
> local OAuth callback. As written it requests the scopes
> `business_management, public_profile, pages_show_list, pages_read_engagement` — it does **not** request
> `ads_read` or `ads_management`, so a token minted that way will likely fail on ad endpoints. Unless you
> patch `AUTH_SCOPE` in `meta_ads_mcp/core/auth.py`, supply a token from the Graph API Explorer instead.

Tokens expire. Long-lived user tokens last about 60 days, so plan to rotate.

---

## 2a. Run locally (stdio)

Best for a single user on their own machine.

```bash
git clone https://github.com/GOFORWRD/meta-ads-mcp.git
cd meta-ads-mcp
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pip install -e .
```

Create a `.env` (see [`.env.example`](.env.example)):

```bash
META_APP_ID=your_app_id
META_APP_SECRET=your_app_secret
META_ACCESS_TOKEN=your_long_lived_token
```

Run it:

```bash
python -m meta_ads_mcp                 # stdio (default) — for MCP clients
python -m meta_ads_mcp --version
```

`META_APP_SECRET` is used to compute `appsecret_proof`, which Meta requires when your app has
"Require app secret" enabled. Tokens are cached at:

| OS | Path |
|---|---|
| macOS | `~/Library/Application Support/meta-ads-mcp/token_cache.json` |
| Linux | `~/.config/meta-ads-mcp/token_cache.json` |
| Windows | `%APPDATA%\meta-ads-mcp\token_cache.json` |

---

## 2b. Run on a VPS (Docker + Caddy)

This is the deployment this fork is actually run on: the server speaks **streamable HTTP**, sits behind
Caddy, and is exposed through a Cloudflare Tunnel so no ports are opened on the host.

```
MCP client ──HTTPS──> Cloudflare Tunnel ──> 127.0.0.1:8789 (Caddy) ──> meta-ads-mcp:8080 (/mcp)
                                                  │
                                         injects Authorization: Bearer <token>
```

### How authentication works here

The server reads the Meta token from an `Authorization: Bearer <token>` header on every request (it also
accepts `X-META-ACCESS-TOKEN`). Rather than making every client send that header, Caddy is configured to:

1. Serve the MCP endpoint only under a **long random secret path prefix**.
2. Strip that prefix and **inject the `Authorization` header** on the way through.
3. Return `404` for anything else.

The result is a single URL that is itself the credential — convenient for MCP clients that can't send custom
headers. Understand the tradeoff before using it; see [Security notes](#security-notes).

### Files

Create these on the VPS **next to the cloned repo**. They are deliberately git-ignored so your live token
never gets committed.

`docker-compose.yml`:

```yaml
services:
  meta-ads-mcp:
    build: .
    container_name: meta-ads-mcp
    restart: unless-stopped
    env_file:
      - .env
    command:
      - "python"
      - "-m"
      - "meta_ads_mcp"
      - "--transport"
      - "streamable-http"
      - "--host"
      - "0.0.0.0"
      - "--port"
      - "8080"
    expose:
      - "8080"
    healthcheck:
      test: ["CMD", "python3", "-c", "import socket; socket.create_connection(('127.0.0.1', 8080), 2)"]
      interval: 30s
      timeout: 5s
      retries: 3

  caddy:
    image: caddy:2-alpine
    container_name: meta-ads-caddy
    restart: unless-stopped
    ports:
      # Loopback only — published to the internet via the Cloudflare Tunnel.
      - "127.0.0.1:8789:80"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
    depends_on:
      - meta-ads-mcp
```

`Caddyfile` — replace `YOUR_SECRET_PATH` with a long random string
(`openssl rand -hex 24`) and `YOUR_META_ACCESS_TOKEN` with your token:

```caddyfile
:80 {
    handle_path /YOUR_SECRET_PATH/* {
        reverse_proxy meta-ads-mcp:8080 {
            header_up Authorization "Bearer YOUR_META_ACCESS_TOKEN"
            # Required for streaming responses — do not buffer, do not time out.
            flush_interval -1
            transport http {
                read_timeout 0
            }
        }
    }

    # Anything without the secret prefix is not found.
    handle {
        respond 404
    }
}
```

`.env` — note there is **no** `META_ACCESS_TOKEN` here, because Caddy supplies it per request:

```bash
META_APP_ID=your_app_id
META_APP_SECRET=your_app_secret
```

### Bring it up

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f meta-ads-mcp
```

Verify Caddy is gating correctly — the bare path must 404, the secret path must not:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8789/          # expect 404
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8789/YOUR_SECRET_PATH/mcp
```

### Expose it with a Cloudflare Tunnel

```bash
cloudflared tunnel login
cloudflared tunnel create meta-ads-mcp
cloudflared tunnel route dns meta-ads-mcp mcp.example.com
```

Point the tunnel's ingress at `http://127.0.0.1:8789`, then run it as a service
(`cloudflared service install`). Your MCP endpoint is:

```
https://mcp.example.com/YOUR_SECRET_PATH/mcp
```

---

## Connecting MCP clients

### Claude Desktop / Claude Code — local stdio

Add to your MCP config (`claude_desktop_config.json`, or `claude mcp add` for Claude Code):

```json
{
  "mcpServers": {
    "meta-ads": {
      "command": "python",
      "args": ["-m", "meta_ads_mcp"],
      "env": {
        "META_APP_ID": "your_app_id",
        "META_APP_SECRET": "your_app_secret",
        "META_ACCESS_TOKEN": "your_long_lived_token"
      }
    }
  }
}
```

### Remote (self-hosted) endpoint

```bash
claude mcp add --transport http meta-ads https://mcp.example.com/YOUR_SECRET_PATH/mcp
```

For clients that support custom headers, you can skip the secret-path trick and send the token yourself:

```
Authorization: Bearer <your_meta_access_token>
```

---

## Configuration reference

### CLI

| Flag | Default | Description |
|---|---|---|
| `--transport` | `stdio` | `stdio` or `streamable-http` |
| `--host` | `localhost` | Bind host (streamable-http only) |
| `--port` | `8080` | Bind port (streamable-http only) |
| `--sse-response` | off | Use SSE framing instead of JSON responses |
| `--login` | — | Run the local Meta OAuth flow (see scope caveat above) |
| `--app-id` | — | Meta App ID for the login flow |
| `--version` | — | Print version |

The HTTP endpoint path is `/mcp`.

### Environment variables

| Variable | Purpose |
|---|---|
| `META_APP_ID` | Meta app ID |
| `META_APP_SECRET` | Meta app secret; enables `appsecret_proof` |
| `META_ACCESS_TOKEN` | Access token, used when no request header supplies one |
| `META_ADS_ENABLE_REPORTS` | Enable `generate_report` |
| `META_ADS_ENABLE_DUPLICATION` | Enable the `duplicate_*` tools |
| `META_ADS_ENABLE_SAVE_AD_IMAGE_LOCALLY` | Enable `save_ad_image_locally` |
| `META_ADS_DISABLE_ADS_LIBRARY` | Disable `search_ads_archive` |
| `META_ADS_DISABLE_LOGIN_LINK` | Suppress login links in tool output |
| `META_ADS_DISABLE_CALLBACK_SERVER` | Don't start the OAuth callback server |
| `META_MCP_DISABLE_DELIVERY_FALLBACK` | Disable the delivery-estimate fallback |
| `META_ADS_OAUTH_HOST` | Alternative OAuth callback hostname |
| `META_ADS_OAUTH_CERT_FILE` / `META_ADS_OAUTH_KEY_FILE` | TLS cert/key for an HTTPS OAuth callback |

Feature-flagged tools are hidden unless their variable is set. See [`.env.example`](.env.example) for the
full annotated list.

---

## Available tools

42 tools are registered. Names below are the actual MCP tool names — some clients (Cursor especially)
display them with an `mcp_meta_ads_` prefix.

**Accounts & pages** — `get_ad_accounts`, `get_account_info`, `get_account_pages`, `search_pages_by_name`

**Campaigns** — `get_campaigns`, `get_campaign_details`, `create_campaign`, `update_campaign`

**Ad sets** — `get_adsets`, `get_adset_details`, `create_adset`, `update_adset`

**Ads** — `get_ads`, `get_ad_details`, `create_ad`, `update_ad`

**Creatives & media** — `get_ad_creatives`, `get_creative_details`, `create_ad_creative`,
`update_ad_creative`, `upload_ad_image`, `get_ad_image`, `get_image_by_hash`, `compute_image_crops`,
`get_ad_video`, `save_ad_image_locally`*

**Insights & reporting** — `get_insights`, `generate_report`*

**Targeting & audiences** — `estimate_audience_size`, `search_interests`, `get_interest_suggestions`,
`search_behaviors`, `search_demographics`, `search_geo_locations`

**Budget** — `create_budget_schedule`

**Duplication*** — `duplicate_campaign`, `duplicate_adset`, `duplicate_ad`, `duplicate_creative`

**Ads Library** — `search_ads_archive`

**Deep research** — `search`, `fetch`

\* Gated behind a feature flag — see [Configuration reference](#configuration-reference).

---

## Security notes

Read [SECURITY.md](SECURITY.md) for the full model. The points that matter most when self-hosting:

- **The secret-path URL is a credential.** With the Caddy setup above, anyone holding the URL has full
  access to your ad accounts. URLs leak more easily than headers — they land in shell history, proxy and
  access logs, and client config files. Use a long random path, rotate it if it is ever exposed, and prefer
  header-based auth (`Authorization: Bearer`) with clients that support it.
- **Never commit `.env`, `Caddyfile`, or any deploy key.** They are in `.gitignore`; keep them there. If you
  add automation that runs `git add -A`, confirm the ignore rules hold before pointing it at a public remote.
- **Bind Caddy to loopback.** The compose file publishes `127.0.0.1:8789` deliberately, so the only public
  route is the tunnel.
- **Scope the token.** Use a token that only has access to the ad accounts you actually need, and rotate it
  on the ~60 day expiry.
- The server strips `access_token` and `appsecret_proof` from URLs before logging.

---

## Testing

```bash
pip install -r requirements.txt
pytest tests/
```

Files ending in `_e2e.py` call the live Meta API and need valid credentials plus real account IDs; the rest
are mocked and run offline. The account IDs in the test suite are placeholders — point them at your own
accounts before running the e2e tests.

---

## Troubleshooting

**`(#200) Requires ads_management permission` / empty account list.** Your token lacks ad permissions. See
the scope caveat in [step 1](#getting-an-access-token) — tokens from the built-in `--login` flow do not
request `ads_read`/`ads_management`.

**`Invalid OAuth access token` or sudden 190 errors.** The token expired (long-lived user tokens last ~60
days) or was invalidated by a password change. Mint a new one and update `.env` or the `Caddyfile`.

**`appsecret_proof` errors.** `META_APP_SECRET` is missing or does not match `META_APP_ID`.

**404 from the tunnel.** Expected on any path without the secret prefix. Check the full URL ends in
`/YOUR_SECRET_PATH/mcp`.

**Streaming responses hang or truncate.** Make sure `flush_interval -1` and `read_timeout 0` are present in
the Caddyfile — without them Caddy buffers and times out long-running tool calls.

**Client connects but lists no tools.** Feature-flagged tools stay hidden until their env var is set; check
the table above.

---

## Licence

Business Source License 1.1 — see [LICENSE](LICENSE). Copyright © 2025 the upstream authors
(ARTELL SOLUÇÕES TECNOLÓGICAS LTDA). Converts to Apache 2.0 on 1 January 2029.
