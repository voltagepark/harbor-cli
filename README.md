# Harbor CLI

Command-line interface for Voltage Park Harbor — authenticate to the Harbor API and pipe access tokens into `curl`, HTTP clients, or shell scripts.

Authentication uses Auth0's [OAuth 2.0 Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628), so no client secret ever lives on your machine. Tokens are cached per (stage, region) under `~/.harbor/` and refreshed automatically before expiry.

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

### Verify (optional)

Confirm the binary runs and reports its version:
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
```bash
# 1. Log into production SEA-1 (default stage + default region).
#    The CLI automatically opens your default browser to complete login.
harbor auth login

# 2. Verify who you are and when the token expires.
harbor auth status

# 3. Pipe the token into curl. `harbor auth token` writes ONLY the raw JWT
#    to stdout, so $(...) never leaks user-facing text into the request.
curl -H "Authorization: Bearer $(harbor auth token)" \
  https://api.sea1.voltagepark.com/v1/fleets
```
```

## Commands

All authentication commands live under `harbor auth`. Global command list:

| Command | Description |
|---------|-------------|
| `harbor auth login [--stage] [--region]` | Log in via the browser (OAuth 2.0 device flow, RFC 8628). |
| `harbor auth token [--stage] [--region]` | Print the current access token to stdout. Auto-refreshes within 5 min of expiry. |
| `harbor auth status [--stage] [--region] [-v]` | Show user, org, expiry, audience. `-v` also prints RBAC permissions. |
| `harbor auth logout [--stage] [--region]` | Clear cached credentials for a stage+region. |
| `harbor version` | Print the CLI version. |
| `harbor help [command]` | Show help for any command. |

### Global flags

Every `harbor auth` subcommand accepts the same two selectors:

| Flag | Default | Values | Purpose |
|------|---------|--------|---------|
| `--stage` | `prod` | `prod`, `beta` | Which Auth0 tenant / environment to target. |
| `--region` | stage default (`sea1`) | `sea1`, `iad1`, `ord1` (region set depends on stage) | Which regional API audience to target. Each access token is scoped to exactly one region. |

Both flags select the (stage, region) key used to look up credentials in `~/.harbor/credentials.json`. They do **not** talk to a server on their own — they just pick which cached token to read, write, or refresh.

### `harbor auth login`

Kicks off the device authorization flow for the selected (stage, region).

```bash
harbor auth login                          # prod / sea1 (default)
harbor auth login --region iad1            # prod / iad1
harbor auth login --stage beta             # beta / sea1
harbor auth login --stage beta --region sea1
```

Walkthrough:

```
$ harbor auth login
Starting device authorization for stage "prod" region "sea1"...

  One-time code: ABCD-EFGH
  Open in browser: https://voltagepark-harbor.us.auth0.com/activate?user_code=ABCD-EFGH

Attempting to open your browser automatically...
Waiting for you to complete the login (Ctrl-C to cancel)...

Logged in as you@voltagepark.com (org b73aedfd-...) -- prod/sea1
Credentials saved to /Users/you/.harbor/credentials.json
```

The CLI tries to open your default browser at the verification URL. On a headless server or SSH session it silently falls back to printing the URL — copy it to any device that has a browser, log in, and the CLI on the original machine picks up the token automatically.

You can be logged into multiple regions simultaneously; each `harbor auth login --region <r>` writes a separate entry keyed by `stage/region`.

### `harbor auth token`

Prints the raw access token for the selected (stage, region) to **stdout** — nothing else. All prompts, refresh notices, and errors go to **stderr**. This is what makes `$(harbor auth token)` safe to embed in a request.

```bash
harbor auth token                          # prod / sea1
harbor auth token --region iad1            # prod / iad1
harbor auth token --stage beta             # beta / sea1
```

If the cached token is within 5 minutes of expiring, `harbor auth token` silently refreshes it (using the stored refresh token) before printing. If the refresh token has been revoked or expired, it prints `refresh failed: invalid_grant` to stderr and exits non-zero — run `harbor auth login` to recover.

### `harbor auth status`

Shows the identity, org, expiry, and audience of the cached token for a given (stage, region). Reads only local state — does not call Auth0 or Harbor.

```bash
harbor auth status                         # prod / sea1
harbor auth status --region iad1
harbor auth status -v                      # also list RBAC permissions
```

Flags:

| Flag | Purpose |
|------|---------|
| `--stage`, `--region` | Which credential entry to inspect. |
| `-v`, `--verbose` | Also print the JWT's `permissions` claim (RBAC scopes). Hidden by default to reduce shoulder-surfing risk. |

Exit code is non-zero when there is no cached credential for the requested (stage, region).

### `harbor auth logout`

Deletes the cached credential entry for one (stage, region). Other regions and stages are untouched.

```bash
harbor auth logout                         # clear prod/sea1
harbor auth logout --region iad1           # clear prod/iad1
harbor auth logout --stage beta            # clear beta/sea1
```

To clear everything at once:

```bash
rm ~/.harbor/credentials.json
```

### `harbor version`

Prints the CLI version string (e.g. `v0.1.0-beta.2`) compiled into the binary.

```bash
harbor version
```

## Stages and regions

An Auth0 access token has exactly one `aud` (audience) claim, so **each token targets exactly one regional Harbor API**. Sending a SEA-1 token to `api.iad1.voltagepark.com` will fail JWT validation — always match `--region` to the host you're calling.

| Stage | Region | API hostname |
|-------|--------|--------------|
| `prod` | `sea1` (default) | `api.sea1.voltagepark.com` |
| `prod` | `iad1` | `api.iad1.voltagepark.com` |
| `prod` | `ord1` | `api.ord1.voltagepark.com` |
| `beta` | `sea1` | `api.sea1.beta.voltagepark.com` |

## Using the token with curl

```bash
# One-shot (defaults to prod/sea1).
curl -H "Authorization: Bearer $(harbor auth token)" \
  https://api.sea1.voltagepark.com/v1/fleets

# Target IAD-1 -- --region must match the API host.
curl -H "Authorization: Bearer $(harbor auth token --region iad1)" \
  https://api.iad1.voltagepark.com/v1/fleets

# Beta environment.
curl -H "Authorization: Bearer $(harbor auth token --stage beta)" \
  https://api.sea1.beta.voltagepark.com/v1/fleets

# Convenience alias for the default region.
alias harbor-curl='curl -H "Authorization: Bearer $(harbor auth token)"'
harbor-curl https://api.sea1.voltagepark.com/v1/fleets
```

## Token storage

Credentials live at `~/.harbor/credentials.json` (mode `0600`, directory `0700`), keyed by `stage/region`:

```json
{
  "credentials": {
    "prod/sea1": {
      "access_token": "eyJ...",
      "refresh_token": "v1.M...",
      "token_type": "Bearer",
      "expires_at": "2026-07-31T04:24:46Z",
      "stage": "prod",
      "region": "sea1"
    },
    "prod/iad1": { "...": "..." }
  }
}
```

Refresh tokens rotate on every use. Access tok
