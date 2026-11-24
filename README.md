# walkley-data for Hermes Agent

A [Portable Agent Plugins v1](https://agent-plugins.org) package for [Hermes Agent](https://github.com/NousResearch/hermes-agent).
It gives a Hermes agent three lookups from a hosted MCP server:

- **China workday calendar:** is a date a working day in mainland China, and how many working days fall in a range.
  It applies the State Council (国务院) holiday notices, including weekend makeup workdays (调休补班). Coverage
  2020-01-01 to 2027-12-31; every answer carries the gov.cn notice URL.
- **Package advisories:** known [OSV.dev](https://osv.dev) advisories (CVE, GHSA, PYSEC, GO, RUSTSEC, OpenSSF
  malicious-package MAL- entries) for one exact npm, PyPI, Go, crates.io or Maven version, with CVSS, affected ranges
  and fixed versions.
- **SEC insider trades:** transactions in the 4 most recent SEC Form 4 / 4/A filings of a US public company, by
  ticker or CIK. Data from [SEC EDGAR](https://www.sec.gov/edgar); not affiliated with or endorsed by the SEC.
  Paid only (see below).

It bundles one MCP server and one skill, and nothing else (no Python, no executables, no local command):

| Part | What it is | Moves money |
|---|---|---|
| `walkley-data` | the hosted MCP server, `https://mvqz0m8d6g.execute-api.ap-northeast-1.amazonaws.com/mcp` (Streamable HTTP, no key, no sign-in) | no |
| `skills/walkley-data` | when to call the tools, and how to treat the answers and payment links | no |

## Install

```bash
hermes plugins install walkley-data   # once listed in Hermes's plugin catalog
hermes plugins enable walkley-data    # portable packages install disabled
```

Before it is listed: `hermes plugins install walkley/walkley-data-hermes-plugin --no-enable`, then enable it.

The tools arrive as `mcp__walkley-data__cn_workday_check`, `mcp__walkley-data__package_advisories` and
`mcp__walkley-data__sec_insider_trades`. All three are annotated `readOnlyHint`.

## Free calls and payment

- `cn_workday_check` and `package_advisories` share 20 free calls per IP address per UTC day (and at most 2,000 free
  calls a day for all callers together). Calls rejected for bad input, such as a date outside the coverage, are not
  counted.
- `sec_insider_trades` has no free calls.
- Past the free calls, and on every `sec_insider_trades` call, the tool answers with an error that names the paid HTTP
  URL for the same lookup and its price, and carries the x402 payment requirements in `structuredContent`. Prices:
  $0.003 for one China date, $0.005 for a date range, $0.005 for one package version, $0.003 for insider trades.
  Payment is USDC on Base (x402 v2, scheme `exact`), made by an x402 client to that URL; it is currently settled
  through the PayAI facilitator. The MCP endpoint itself never verifies or settles a payment.
- This plugin has no wallet and cannot pay. The skill tells the agent never to pay without the user's consent.

## Disclosure

- **Network calls:** to `https://mvqz0m8d6g.execute-api.ap-northeast-1.amazonaws.com/mcp` only (an AWS API Gateway
  endpoint run by the plugin author), and only when the agent calls one of its tools. The server receives the tool's
  arguments: dates, package ecosystem, name and version, a ticker or CIK. For advisories it queries OSV.dev; for China
  dates it reads its own bundled data. `sec_insider_trades` on this endpoint does not call SEC.
- **What the tools do:** three lookups, annotated `readOnlyHint`. No tool signs, sends a transaction, holds or asks for
  a private key, or moves money. The server is marked `untrusted` in `plugin.json`, so any tool the server lists
  without `readOnlyHint` would need the user's approval.
- **No credentials:** the plugin stores and reads none.
- **No local side effects:** no shell commands, no background processes, no files read or written outside Hermes.
- **No telemetry from the plugin.** On the server, the free-call counters store no raw IP address, only a keyed hash
  (HMAC-SHA256 of the day and the IP) deleted after 2 days; request logs keep only the same hash, which changes every
  UTC day; API Gateway access logs keep the source IP for 30 days.
- **Not covered by the pin:** the tool set and prices are served by the hosted server at runtime. The server's source is
  public at [walkley/x402-content-site](https://github.com/walkley/x402-content-site) (`src/mcp.ts`).

## Without the plugin

Add the server by hand to `~/.hermes/config.yaml`, then run `/reload-mcp`:

```yaml
mcp_servers:
  walkley-data:
    url: "https://mvqz0m8d6g.execute-api.ap-northeast-1.amazonaws.com/mcp"
```

Data only, not legal, tax or investment advice. MIT.
