# Harbor CLI

Command-line interface for [Voltage Park Harbor](https://voltagepark.com) — authenticate to the Harbor API and pipe access tokens into `curl`, HTTP clients, or shell scripts.

> **Pre-release: not yet supported for external customers.** APIs, flags, and credential storage may change without notice until we cut `v1.0.0`. For access to the private beta program, contact your Voltage Park representative.

## Install

### Direct download

**macOS (Apple Silicon):**

```bash
curl -fsSL https://github.com/voltagepark/harbor-cli/releases/latest/download/harbor-darwin-arm64 -o harbor && sudo install -m 755 harbor /usr/local/bin/harbor && rm harbor
```

**macOS (Intel):**

```bash
curl -fsSL https://github.com/voltagepark/harbor-cli/releases/latest/download/harbor-darwin-amd64 -o harbor && sudo install -m 755 harbor /usr/local/bin/harbor && rm harbor
```

**Linux (x86_64):**

```bash
curl -fsSL https://github.com/voltagepark/harbor-cli/releases/latest/download/harbor-linux-amd64 -o harbor && sudo install -m 755 harbor /usr/local/bin/harbor && rm harbor
```

**Linux (ARM64):**

```bash
curl -fsSL https://github.com/voltagepark/harbor-cli/releases/latest/download/harbor-linux-arm64 -o harbor && sudo install -m 755 harbor /usr/local/bin/harbor && rm harbor
```

**Windows (x86_64, PowerShell):**

```powershell
curl.exe -sSL https://github.com/voltagepark/harbor-cli/releases/latest/download/harbor-windows-amd64.exe -o harbor.exe
```

**Windows (ARM64, PowerShell):**

```powershell
curl.exe -sSL https://github.com/voltagepark/harbor-cli/releases/latest/download/harbor-windows-arm64.exe -o harbor.exe
```

### Verify

```bash
harbor version
```

Optional checksum verification (each release ships a `checksums.txt`):

```bash
VERSION=$(harbor version)
curl -sSL https://github.com/voltagepark/harbor-cli/releases/download/${VERSION}/checksums.txt \
  | shasum -a 256 -c --ignore-missing
```

## Quick start

```bash
# Log into production SEA-1 (default stage + region)
harbor auth login

# Print the current access token to stdout and pipe into curl
curl -H "Authorization: Bearer $(harbor auth token)" \
  https://api.sea1.voltagepark.com/v1/fleets
```

## Commands

| Command | Description |
|---------|-------------|
| `harbor auth login [--stage] [--region]` | Log in via the browser (OAuth 2.0 device flow, RFC 8628). |
| `harbor auth token [--stage] [--region]` | Print the current access token to stdout. Auto-refreshes when nearing expiry. |
| `harbor auth status [--stage] [--region] [-v]` | Show user, org, expiry. `-v` also prints RBAC permissions. |
| `harbor auth logout [--stage] [--region]` | Clear cached credentials for a stage+region. |
| `harbor version` | Print the CLI version. |

## Stages and regions

| Stage | Region | API hostname |
|-------|--------|--------------|
| `prod` | `sea1` (default) | `api.sea1.voltagepark.com` |
| `prod` | `iad1`            | `api.iad1.voltagepark.com` |
| `prod` | `ord1`            | `api.ord1.voltagepark.com` |
| `beta` | `sea1`            | `api.sea1.beta.voltagepark.com` |

`--stage` defaults to `prod`. `--region` defaults to `sea1`.

Each cached token targets exactly one regional API — a SEA-1 token cannot be used against IAD-1 or ORD-1. Log in per region as needed; tokens coexist under `~/.harbor/credentials.json`.

## Token storage

Credentials live at `~/.harbor/credentials.json` (mode `0600`, directory `0700`), keyed by `stage/region`. Refresh tokens rotate on every use; access tokens are silently refreshed within 5 minutes of expiry.

## Security notes

- The Auth0 Client ID embedded in the binary is a **public identifier** (Native App, no client secret).
- `harbor auth token` writes only the raw JWT to stdout; prompts and status go to stderr, so `$(harbor auth token)` never leaks user-facing text.
- `harbor auth status` hides RBAC permissions by default — pass `-v` to see them.
- Signature verification is performed by the Harbor API, not the CLI.

## License

[Apache-2.0](LICENSE)

## Source

Source code is maintained privately in `voltagepark/harbor-service`. Only compiled binaries are published here. For access to the private beta program, contact your Voltage Park representative.
