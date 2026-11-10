# patchbay MCP audit

GitHub Action that audits a Model Context Protocol (MCP) server for OAuth 2.1 and security gaps. 16 checks covering RFC 8707 audience validation, RFC 8252 loopback redirect URIs, OAuth metadata, scope minimisation, secret surface and tool manifest drift.

No backend, no account. Wraps the open-source `@patchbay-nebius/agent` CLI.

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

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `url` | yes | (none) | MCP server URL to audit |
| `fail-on` | no | `high` | Fail on this severity or higher: `critical`, `high`, `medium`, `low`, `info` |
| `auth` | no | (none) | Bearer token for authenticated servers |
| `probe-audience` | no | (none) | Bearer token for a different resource on the same AS. Enables the AUDIENCE-001 cross-audience check (RFC 8707) |
| `probe-dcr-loopback` | no | `false` | Opt into AUTH-006 active DCR loopback probe (RFC 8252 §7.3). Writes to the AS |
| `sarif-output` | no | (none) | If set, also write a SARIF report there (upload with `github/codeql-action/upload-sarif`) |
| `json-output` | no | (none) | If set, also write a JSON report there |
| `agent-version` | no | latest | Pin a specific `@patchbay-nebius/agent` version |

## Uploading SARIF to code scanning

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

Findings land in the Security tab of your repo.

## Authenticated servers

Store the bearer token as a repository secret, then:

```yaml
      - uses: GodSmasher/patchbay-audit@v1
        with:
          url: https://your-mcp-server.example.com/mcp
          auth: ${{ secrets.MCP_AUDIT_TOKEN }}
```

## What is audited

- **OAuth 2.1.** Auth server metadata, token endpoint, DCR, loopback redirect URIs (RFC 8252 §7.3), audience validation (RFC 8707), scope posture.
- **Transport.** Streamable HTTP conformance, origin validation.
- **Secrets.** Token or API-key exposure in tool definitions.
- **Manifest.** Tool list drift detection (pin on first scan, diff on every later run).
- **Probes.** Reachability, auth posture judgement.

Full list: see [checks](https://github.com/GodSmasher/patchbay/tree/main/packages/audit-core/src/checks) in the main repo.

## Links

- Main repo and CLI: [github.com/GodSmasher/patchbay](https://github.com/GodSmasher/patchbay)
- Hosted scanner (no install): [patchbay-mcp.dev/scan](https://patchbay-mcp.dev/scan)
- npm: [@patchbay-nebius/agent](https://www.npmjs.com/package/@patchbay-nebius/agent)

## License

MIT
