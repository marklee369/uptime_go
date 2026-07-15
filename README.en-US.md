# Uptime

Uptime is a lightweight, self-hosted Linux monitoring system written in Go. It includes a server (`uptimed`), an agent (`uptime-agent`), and a responsive Chart.js web dashboard.

## Features

- CPU, memory, disk, network, process, connection, load, and uptime monitoring
- Latest status, compact binary history, charts, and offline agent queue
- HMAC-SHA256 authentication with timestamp and nonce replay protection
- Gotify alerts, systemd services, and `/healthz`
- Linux amd64/arm64 support; no database or Node.js server runtime
- The agent has no listener, remote shell, or command execution endpoint

## Install

Run as root. The installers download the matching binary from the **latest GitHub Release** by default and verify it against that release's `SHA256SUMS`.

Install the server:

```sh
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-server.sh | sh
```

Add a node and restart the server:

```sh
su -s /bin/sh uptime -c '/usr/local/bin/uptimed -config /etc/uptime/server.json -add-node "US VPS 01"'
systemctl restart uptimed
```

Install the agent using the registration code:

```sh
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-agent.sh | sh -s -- https://status.example.com
```

The installer reads the registration code without echoing it, so it is not stored in shell history or exposed as a process argument. For unattended installation, place the code in a root-only file and set `REGISTRATION_CODE_FILE=/path/to/code`.

Defaults:

- Server: `127.0.0.1:62369`
- Server data: `/opt/uptime`
- Agent spool: `/var/lib/uptime-agent/spool.jsonl`
- Agent interval: 60 seconds

Custom installation examples:

```sh
# Server port
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-server.sh | PORT=8080 sh

# Server data directory
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-server.sh | DATA_DIR=/srv/uptime-data sh

# Agent spool directory
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-agent.sh | DATA_DIR=/srv/uptime-agent-data sh -s -- https://status.example.com
```

Use an HTTPS reverse proxy in production. Plain HTTP agent connections are accepted only for literal `127.0.0.1` or `::1` addresses. Keep registration codes private.

## Update

Running an installer again prompts before updating and preserves the existing configuration, server history, and agent spool. For unattended updates:

```sh
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-server.sh | REINSTALL=1 sh
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-agent.sh | REINSTALL=1 sh
```

The default `VERSION=latest` follows the newest Release and is not pinned to a specific version. Set `VERSION` to a release tag only when an explicit version is required.

## Configuration

Examples are available in [`config.example.json`](config.example.json), [`agent.example.json`](agent.example.json), and [`systemd/`](systemd/).

- Server configuration: `/etc/uptime/server.json`, mode `0600`
- Agent configuration: `/etc/uptime/agent.json`, mode `0600`
- `offline_after_seconds`: 5–86400 seconds, default 180
- `history_days`: retained history duration
- `trusted_proxies`: proxy IP/CIDR allowlist used before accepting `CF-Connecting-IP`, `X-Real-IP`, or `X-Forwarded-For`; the proxy must overwrite these headers
- `country_code`: optional manual country override
- `CF-IPCountry`: used automatically when requests pass through Cloudflare

Do not run multiple server instances against the same data directory. Restart `uptimed` after editing its configuration or adding nodes.

## Gotify Alerts

Configure `gotify.url`, `gotify.token`, and `gotify.priority` in `server.json`. Gotify must use HTTPS except for loopback testing.

Alerts are deduplicated for:

- CPU at or above 50% for 10 minutes
- Memory at or above 90%
- Sustained upload above the configured threshold for 24 hours
- Nodes exceeding `offline_after_seconds`

Use `alerts.disable_upload` or `alerts.upload_min_mib_per_second` to adjust upload alerts.

## Build and Release

Go 1.24 or newer is required:

```sh
make test
make build
```

Pushing a semantic version tag runs tests and publishes Linux amd64/arm64 binaries with checksums:

```sh
git tag v1.4.0
git push origin v1.4.0
```

Release assets are published to the public [`marklee369/uptime_go`](https://github.com/marklee369/uptime_go) repository and cannot be overwritten by the release workflow. Verify downloaded binaries and installers against the accompanying `SHA256SUMS`.

The private source repository must define an Actions secret named `UPTIME_GO_TOKEN`. Use a fine-grained token with **Contents: Read and write** access to `marklee369/uptime_go`, and ensure the public repository has an initialized default branch.

## API and Security

- `POST /api/v1/ingest` — signed agent reports
- `GET /api/v1/overview` — latest public node status
- `GET /api/v1/nodes/{id}/history?since=UNIX&until=UNIX` — node history
- `GET /api/v1/status` — monitoring server status
- `GET /healthz` — health check

Ingest requests use HMAC-SHA256, a five-minute timestamp window, and one-time nonces. If clocks differ, the agent learns and reuses a server-time offset while preserving queued samples' relative timestamps. The server clock never moves backwards during a process lifetime, but NTP is still required for TLS validation and accurate wall-clock history. Public responses use explicit field allowlists, cross-site browser API requests are rejected, node IDs are validated, and history paths cannot be selected directly by user input.

## Credits

The terminal-style information architecture is inspired by [CF-Server-Monitor](https://github.com/huilang-me/CF-Server-Monitor) and [LTstats](https://github.com/lukastautz/ltstats). Charts use [Chart.js](https://www.chartjs.org). The Go server, agent, storage, and frontend implementation are independent.

## Uninstall

Warning: these commands permanently delete server history or the agent spool. Replace the default data directory if you customized `DATA_DIR`.

Remove the server:

```sh
systemctl disable --now uptimed.service 2>/dev/null || true
rm -f /etc/systemd/system/uptimed.service
rm -f /etc/systemd/system/multi-user.target.wants/uptimed.service
rm -rf /etc/systemd/system/uptimed.service.d
rm -f /usr/local/bin/uptimed /etc/uptime/server.json
rm -rf /opt/uptime
userdel uptime 2>/dev/null || true
systemctl daemon-reload
systemctl reset-failed uptimed.service 2>/dev/null || true
if [ ! -e /etc/uptime/agent.json ]; then rmdir /etc/uptime 2>/dev/null || true; fi
```

Remove the agent:

```sh
systemctl disable --now uptime-agent.service 2>/dev/null || true
rm -f /etc/systemd/system/uptime-agent.service
rm -f /etc/systemd/system/multi-user.target.wants/uptime-agent.service
rm -rf /etc/systemd/system/uptime-agent.service.d
rm -f /usr/local/bin/uptime-agent /etc/uptime/agent.json
rm -rf /var/lib/uptime-agent
userdel uptime-agent 2>/dev/null || true
systemctl daemon-reload
systemctl reset-failed uptime-agent.service 2>/dev/null || true
if [ ! -e /etc/uptime/server.json ]; then rmdir /etc/uptime 2>/dev/null || true; fi
```
