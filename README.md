# patchbay MCP audit

GitHub Action that audits a Model Context Protocol (MCP) server for OAuth 2.1 and transport-security gaps. 25+ checks covering RFC 8707 audience validation, RFC 8252 loopback redirect URIs, OAuth metadata, scope minimisation, transport security headers, prompt-injection surface in tool descriptions, headless OAuth conformance, and tool-manifest drift.

No backend, no account. Wraps the open-source `@patchbay-nebius/agent` CLI and ships findings as SARIF for GitHub code scanning.

## Quick start

```yaml
name: mcp-audit
on: [push, pull_request]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: GodSmasher/patchbay-audit@v1
        with:
          url: https://your-mcp-server.example.com/mcp
          fail-on: high
```

That fails the job when any `high` or `critical` finding appears.

## Try it against our demo

We host an intentionally vulnerable MCP server for exactly this. Point the Action at it to see what each finding looks like before you wire it into your own CI.

```yaml
      - uses: GodSmasher/patchbay-audit@v1
        with:
          url: https://demo.patchbay-mcp.dev/mcp
          probe-headless-oauth: 'true'
          fail-on: critical
```

Produces 14 findings (4 critical, 5 high, 2 medium, 2 low, 1 info) in under 3 seconds. Live server + bug matrix at [demo.patchbay-mcp.dev](https://demo.patchbay-mcp.dev).

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `url` | yes | (none) | MCP server URL to audit |
| `fail-on` | no | `high` | Fail on this severity or higher: `critical`, `high`, `medium`, `low`, `info` |
| `auth` | no | (none) | Bearer token for authenticated servers |
| `probe-audience` | no | (none) | Bearer token for a different resource on the same AS. Enables the AUDIENCE-001 cross-audience check (RFC 8707) |
| `probe-missing-aud` | no | (none) | Bearer token whose JWT payload has no `aud` claim. Enables the AUDIENCE-002 missing-audience check (catches the `if (aud && aud !== me)` early-exit bug) |
| `probe-dcr-loopback` | no | `false` | Opt into AUTH-006 active DCR loopback probe (RFC 8252 §7.3). Writes to the AS |
| `probe-headless-oauth` | no | `false` | Opt into the headless OAuth conformance battery (PKCE-001, PKCE-002, IMPLICIT-001, REDIRECT-002). Registers a client via DCR and sends four crafted `/authorize` requests. **Only run against servers you own or against the patchbay demo** |
| `sarif-output` | no | (none) | If set, also write a SARIF report there (upload with `github/codeql-action/upload-sarif`) |
| `json-output` | no | (none) | If set, also write a JSON report there |
| `agent-version` | no | latest | Pin a specific `@patchbay-nebius/agent` version |

## Uploading SARIF to GitHub code scanning

```yaml
      - uses: GodSmasher/patchbay-audit@v1
        with:
          url: https://your-mcp-server.example.com/mcp
          sarif-output: patchbay.sarif

      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: patchbay.sarif
```

Findings land in the **Security → Code scanning** tab of your repo as individual alerts, one per finding, with the full fix text from each check.

## Authenticated servers

Store the bearer token as a repository secret, then:

```yaml
      - uses: GodSmasher/patchbay-audit@v1
        with:
          url: https://your-mcp-server.example.com/mcp
          auth: ${{ secrets.MCP_AUDIT_TOKEN }}
```

## What is audited

- **OAuth 2.1 metadata.** Authorization server metadata (RFC 8414), PKCE S256, DCR / CIMD registration, loopback redirect URIs (RFC 8252 §7.3), RFC 9207 `iss` parameter.
- **OAuth 2.1 behaviour (opt-in headless battery).** Server accepts a `/authorize` request with no `code_challenge` (PKCE-001), with `code_challenge_method=plain` (PKCE-002), with `response_type=token` implicit (IMPLICIT-001), or with an unregistered `redirect_uri` (REDIRECT-002).
- **Audience validation (opt-in with two tokens).** Cross-audience acceptance (AUDIENCE-001, RFC 8707) and missing-`aud` early-exit (AUDIENCE-002).
- **Protected resource.** Protected Resource Metadata presence and `authorization_servers` field (RESOURCE-001, RFC 9728).
- **Scope hygiene.** Per-tool scopes required, read/write scope separation.
- **Transport.** TLS, Streamable HTTP conformance, origin header validation, DNS rebinding guard.
- **Response headers.** HSTS, X-Frame-Options / CSP frame-ancestors, X-Content-Type-Options, CORS misconfigurations.
- **Prompt-injection surface.** Hidden Unicode (zero-width, bidi overrides), role-marker patterns in tool descriptions (POISONING-001).
- **Secrets.** Token / API-key exposure in tool definitions and source.
- **Manifest drift.** Pin the tool list on first scan, flag additions / removals / schema narrowings on every later run.

Full list: see [checks](https://github.com/GodSmasher/patchbay/tree/main/packages/audit-core/src/checks) in the main repo.

## Links

- Main repo and CLI: [github.com/GodSmasher/patchbay](https://github.com/GodSmasher/patchbay)
- Live demo server (scan it yourself): [demo.patchbay-mcp.dev](https://demo.patchbay-mcp.dev)
- Hosted scanner (no install): [patchbay-mcp.dev/scan](https://patchbay-mcp.dev/scan)
- Public MCP server leaderboard: [patchbay-mcp.dev/leaderboard](https://patchbay-mcp.dev/leaderboard)
- npm: [@patchbay-nebius/agent](https://www.npmjs.com/package/@patchbay-nebius/agent)

## License

MIT
