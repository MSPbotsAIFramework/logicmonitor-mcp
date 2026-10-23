# logicmonitor-mcp

**LogicMonitor** MCP server — exposes the LogicMonitor REST API (v3, `/santaba/rest`) as MCP tools.

<div align="center">

## MSPbots Connector

**The MCP Gateway built for MSPs: visible, controlled, and your data stays where it belongs.**

[**Get started at connect.mspbots.ai →**](https://connect.mspbots.ai)

</div>

---

MSPbots Connector gives your AI clients one MCP URL to reach your entire MSP stack (RMM, PSA, documentation, security, finance), and gives you control over who can call which tool and a record of everything that was called.

**Hosted service: [connect.mspbots.ai](https://connect.mspbots.ai)** · Free to use.

### Why Connector

![MSPbots Connector](https://stgoss.mspbots.ai/files/mcp/mspbots-mcp.png)

| | |
|---|---|
| **Visible** | Every tool call is logged: who, which vendor, when. Usage dashboard, CSV export, SIEM streaming. |
| **Controlled** | Role-based, per-tool allowlist. Deny by default. |
| **Protected** | Credentials encrypted per record. AI clients never see your vendor passwords or tokens. |
| **MSP-native** | Built for the MSP stack only: RMM, PSA, documentation, security, finance. |

### Quick start

1. **Sign up** at [connect.mspbots.ai](https://connect.mspbots.ai).
2. **Connect your vendors.** Pick from 150+ connectors and authorize each one on the vendor's own page, or bring your own API keys. No extra vendor accounts needed.
3. **Set access.** Create roles and decide which tools each role may call. Nothing is allowed until you allow it.
4. **Add your MCP URL to your AI client.** Copy the URL from your Connector dashboard into Claude, ChatGPT, Cursor, VS Code, Gemini, Codex, or any MCP-capable client.
5. **Ask your AI to do real work**, and watch every call appear in the audit log.

### Features

**Unified gateway**
- One MCP URL for all your AI clients
- 150+ connectors across the MSP stack
- Custom MCP servers supported, free
- Authorization happens on each vendor's own page; the AI never sees passwords or tokens
- Bring your own credentials: use the vendor API keys you already have
- Multiple accounts per app; connect once, reuse across AI clients
- Tool search: finds the right action for the task
- One sign-in; vendor credentials are injected behind the scenes and OAuth tokens refresh automatically

**Visibility**
- Audit log for every tool call (who, which vendor, when)
- Usage dashboard: call volume, top vendors, error rate, per-user activity
- Export audit logs to CSV, or stream to Loki, Graylog, or LogScale (SIEM)
- Optional recording of tool arguments and result summaries (off by default, enabled per organization)

**Control**
- Per-role, per-tool allowlist, deny by default
- owner / admin / member roles
- SCIM provisioning with Entra ID, Okta, JumpCloud, and Google Workspace

**Security**
- Credentials encrypted one record at a time (AES-256-GCM, per-record key)
- AI clients authenticate with OAuth 2.1 + PKCE
- Vendor responses are scanned for prompt-injection patterns and returned as structured data. This is one layer of defense, not a guarantee against every attack.
- Static egress IP for customer firewall and API allowlisting

### Pricing

**Free.**

Links: [connect.mspbots.ai](https://connect.mspbots.ai) · [mspbots.ai](https://mspbots.ai)

---

## Overview

This server implements the [Model Context Protocol](https://modelcontextprotocol.io/) (Streamable HTTP/SSE transport) and wraps the LogicMonitor endpoints used by MSPbots' own LogicMonitor integration. It follows the MSPbots **Vendor MCP Service SOP**: stateless, no stored credentials, per-request header authentication.

The underlying API authenticates via **LMv1** — a per-request HMAC-SHA256 signature computed from an Access Id / Access Key pair (the same scheme MSPbots itself uses for this integration: Company + Access Id + Access Key, no OAuth). This server computes the LMv1 signature itself for every outgoing request from the credentials supplied in that request's headers; it never persists them.

## Quick Start

### Docker (recommended)

```bash
docker compose up --build
```

The server starts on `http://localhost:8080`.

### Local (uv)

```bash
uv sync
python -m logicmonitor_mcp
```

## Health Check

```bash
curl http://localhost:8080/health
# {"status": "ok"}
```

No credentials are required for the health endpoint.

## Authentication parameters (Authentication)

Every request to `/mcp` must include the following HTTP headers:

| Header | Type | Required | Default | Allowed values | Description | Example |
|---|---|---|---|---|---|---|
| `X-LogicMonitor-Company` | string | Yes | None | None (free text) | LogicMonitor portal subdomain, i.e. the `{company}` part of `https://{company}.logicmonitor.com`; it is also part of the base URL in the signing algorithm. | `X-LogicMonitor-Company: acme` |
| `X-LogicMonitor-Access-Id` | string | Yes | None | None (free text) | Access Id of the LMv1 API Token; same as the "Access Id" field in the MSPbots integration configuration. | `X-LogicMonitor-Access-Id: abcd1234efgh5678` |
| `X-LogicMonitor-Access-Key` | string | Yes | None | None (free text, sensitive) | Access Key (secret) of the LMv1 API Token, used to sign each request with HMAC-SHA256; this service uses it only during the lifetime of the current request and does not persist it. | `X-LogicMonitor-Access-Key: <access_key>` |

Missing any of the three headers returns `401 Unauthorized`.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `MCP_HTTP_PORT` | `8080` | Listening port |
| `MCP_HTTP_HOST` | `0.0.0.0` | Listening host |

logicmonitor-mcp has no required environment variables — all identity (company/access id/access key) is per-request via headers.

## MCP Endpoint

```
POST http://localhost:8080/mcp
```

Connect your MCP client with:
- Transport: `http` (Streamable HTTP / SSE)
- Headers: `X-LogicMonitor-Company`, `X-LogicMonitor-Access-Id`, `X-LogicMonitor-Access-Key` (all required)

## Tool List

**13 tools.** Originally matched the 15 LogicMonitor API endpoints registered under this integration in MSPbots (`sys_integration` subject_code `LOGICMONITOR`); `get_device_datasource_instances` and `get_device_datasource_instance_alertsettings` were removed after intent-matching testing found their instance-ID drill-down (device → DataSource → instance → alertsettings) added confusion without a clear caller need — the remaining `get_device_datasource_data` still covers actual collected metrics. All tools are read-only (`readOnlyHint=True`); there are no write/delete tools in this service.

Pagination: `size` defaults to **50** and is server-side clamped to a hard cap of **1000** — LogicMonitor's own documented/enforced maximum for this parameter (values above 1000 are silently clamped by LogicMonitor itself; requesting more never errors, it just returns at most 1000). Independent of that, any single tool response is capped at 20,000 characters — if a page of results would exceed it, the response is truncated with `truncated`/`original_count` fields rather than returned in full.

### Devices (3)

| Tool | Function | Parameters |
|---|---|---|
| `logicmonitor_get_devices` | List monitored devices | `size=50` (max 1000), `offset=0`, `sort?`, `filter?`, `fields?` |
| `logicmonitor_get_device_groups` | List device groups | `size=50` (max 1000), `offset=0`, `sort?`, `filter?`, `fields?` |
| `logicmonitor_get_device_properties` | Get a device's custom/system properties | `device_id`, `size=50` (max 1000), `offset=0`, `filter?` |

### Device DataSources (2)

| Tool | Function | Parameters |
|---|---|---|
| `logicmonitor_get_device_datasources` | List DataSources applied to a device | `device_id`, `size=50` (max 1000), `offset=0`, `filter?`, `fields?` |
| `logicmonitor_get_device_datasource_data` | Get data points collected by a DataSource | `device_id`, `source_id`, `start?`, `end?` |

### Alerts (3)

| Tool | Function | Parameters |
|---|---|---|
| `logicmonitor_get_alerts` | List active/historical alerts | `size=50` (max 1000), `offset=0`, `sort?`, `filter?`, `fields?` |
| `logicmonitor_get_alert_detail` | Get details of a single alert | `alert_id` |
| `logicmonitor_get_alert_rules` | List alert escalation rules | `size=50` (max 1000), `offset=0`, `filter?` |

### Admin (2)

| Tool | Function | Parameters |
|---|---|---|
| `logicmonitor_get_users` | List portal user (admin) accounts | `size=50` (max 1000), `offset=0`, `filter?`, `fields?` |
| `logicmonitor_get_roles` | List roles (permission sets) | `size=50` (max 1000), `offset=0`, `filter?` |

### Reports (2)

| Tool | Function | Parameters |
|---|---|---|
| `logicmonitor_get_reports` | List reports | `size=50` (max 1000), `offset=0`, `filter?`, `fields?` |
| `logicmonitor_get_report_groups` | List report groups | `size=50` (max 1000), `offset=0`, `filter?` |

### SDT (1)

| Tool | Function | Parameters |
|---|---|---|
| `logicmonitor_get_sdts` | List Scheduled Down Time (SDT) entries | `size=50` (max 1000), `offset=0`, `filter?` |

## Error Handling

Tool errors are returned as a JSON string (not a protocol-level exception), so an Agent can branch on it programmatically:

```json
{"error": {"code": "upstream_error", "message": "...", "retryable": true}}
```

`code` is one of a fixed vocabulary: `not_configured`, `unauthorized`, `not_found`, `invalid_argument`, `rate_limited`, `upstream_error`. `retryable` indicates whether the same call is safe to retry (true for `rate_limited`/`upstream_error`, false otherwise).

## Network Robustness

- Outbound timeouts: connect 5s / read 30s / write 10s / pool 5s.
- Limited retry (up to 3 attempts) with capped exponential backoff on `429`/`5xx`, honoring `Retry-After` when present.
- Each retry attempt recomputes the LMv1 signature from scratch (it embeds the current epoch), so a retried request is never sent with a stale/expired signature.
- A single shared `httpx.AsyncClient` connection pool is reused for the process lifetime.

## Test Example

List devices:

```json
{
  "method": "tools/call",
  "params": { "name": "logicmonitor_get_devices", "arguments": { "size": 10 } }
}
```

Equivalent `curl` against the running server (streamable HTTP MCP endpoint):

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "X-LogicMonitor-Company: <company>" \
  -H "X-LogicMonitor-Access-Id: <access_id>" \
  -H "X-LogicMonitor-Access-Key: <access_key>" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": { "name": "logicmonitor_get_devices", "arguments": { "size": 10 } }
  }'
```

A parameterized call:

```json
{
  "method": "tools/call",
  "params": {
    "name": "logicmonitor_get_device_datasources",
    "arguments": { "device_id": "7186" }
  }
}
```

## API Reference

- Official docs: [LogicMonitor REST API v3](https://www.logicmonitor.com/support/rest-api-developers-guide/overview/using-logicmonitors-rest-api)
- Auth scheme: [LMv1 API Tokens](https://www.logicmonitor.com/support/rest-api-developers-guide/overview/using-logicmonitors-rest-api#Authentication-Tokens)
- Base URL pattern: `https://{company}.logicmonitor.com/santaba/rest`

## Known Gaps / Not Yet Verified

- Not yet tested against a live LogicMonitor portal — only protocol-level verification (health check, 401 on missing/partial headers, `tools/list` returning all 13 tools, unit-tested error-code mapping) has been done so far.
- Tool coverage originally matched MSPbots' existing 15 registered LogicMonitor API entries; 2 were removed post intent-matching testing (see Tool List above). Coverage does not extend to the full LogicMonitor REST API surface (which additionally covers write operations, dashboards, websites, collectors, and LogicModules) — expand on request if broader coverage is needed.
