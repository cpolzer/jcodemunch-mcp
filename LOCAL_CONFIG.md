# Local Configuration Notes

Running jcodemunch-mcp as a single shared service over HTTP transport so multiple Claude Code sessions (and other MCP clients) share one index cache, watcher, and in-memory LRU.

---

## Development Install (Editable Mode)

Installs jcodemunch-mcp into the existing pipx venv pointing at the local source tree.
Any edits to the repo are reflected immediately — no reinstall needed.

```bash
pipx install --editable /path/to/jcodemunch-fork
```

To include optional extras (e.g. HTTP transport, AI summaries):
```bash
pipx install --editable "/path/to/jcodemunch-fork[http,anthropic]"
```

To verify the installed binary points at the local source:
```bash
pipx list --short | grep jcodemunch
cat ~/.local/bin/jcodemunch-mcp | head -3
```

The shebang should reference the pipx venv Python and import from `jcodemunch_mcp.server`.

---

## Linux

### Binary location
```
~/.local/bin/jcodemunch-mcp
```

Editable install from the local repo:
```bash
pipx install --editable /path/to/jcodemunch-fork
```

### Token — GNOME Keyring / KWallet (`secret-tool`)

```bash
# generate
openssl rand -hex 32

# store (prompts for value)
secret-tool store --label="jcodemunch-mcp token" service jcodemunch-mcp username http-token

# retrieve
secret-tool lookup service jcodemunch-mcp username http-token
```

### systemd user service

`~/.config/systemd/user/jcodemunch-mcp.service`:
```ini
[Unit]
Description=jCodemunch MCP Server
After=network.target

[Service]
ExecStart=%h/.config/jcodemunch-mcp/start.sh
Environment=JCODEMUNCH_TRANSPORT=streamable-http
Restart=on-failure

[Install]
WantedBy=default.target
```

`~/.config/jcodemunch-mcp/start.sh` (chmod 700):
```bash
#!/bin/bash
export JCODEMUNCH_HTTP_TOKEN=$(secret-tool lookup service jcodemunch-mcp username http-token)
exec "$HOME/.local/bin/jcodemunch-mcp" serve --transport streamable-http
```

```bash
systemctl --user enable --now jcodemunch-mcp
```

### Populate `settings.local.json` from keyring

```bash
token=$(secret-tool lookup service jcodemunch-mcp username http-token)
jq --arg t "$token" '.mcpServers.jcodemunch.headers.Authorization = "Bearer \($t)"' \
  ~/.claude/settings.local.json > /tmp/s.json && mv /tmp/s.json ~/.claude/settings.local.json
```

---

## macOS

### Binary location
```
~/.local/bin/jcodemunch-mcp   # pipx default
```

Editable install from the local repo:
```bash
pipx install --editable /path/to/jcodemunch-fork
```

### Token — Keychain (`security`)

```bash
# generate
openssl rand -hex 32

# store
security add-generic-password -a http-token -s jcodemunch-mcp -w <token>

# retrieve
security find-generic-password -a http-token -s jcodemunch-mcp -w
```

### launchd user agent

`~/Library/LaunchAgents/com.jcodemunch.mcp.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.jcodemunch.mcp</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>/Users/Shared/jcodemunch-mcp/start.sh</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
</dict>
</plist>
```

`~/Library/Application Support/jcodemunch-mcp/start.sh` (chmod 700):
```bash
#!/bin/bash
export JCODEMUNCH_HTTP_TOKEN=$(security find-generic-password -a http-token -s jcodemunch-mcp -w)
exec "$HOME/.local/bin/jcodemunch-mcp" serve --transport streamable-http
```

```bash
launchctl load ~/Library/LaunchAgents/com.jcodemunch.mcp.plist
```

### Populate `settings.local.json` from keyring

```bash
token=$(security find-generic-password -a http-token -s jcodemunch-mcp -w)
jq --arg t "$token" '.mcpServers.jcodemunch.headers.Authorization = "Bearer \($t)"' \
  ~/.claude/settings.local.json > /tmp/s.json && mv /tmp/s.json ~/.claude/settings.local.json
```

---

## MCP client config (all platforms)

In `.claude/settings.local.json` (gitignored — never commit secrets):
```json
{
  "mcpServers": {
    "jcodemunch": {
      "type": "http",
      "url": "http://127.0.0.1:8901",
      "headers": {
        "Authorization": "Bearer <your-token>"
      }
    }
  }
}
```

Claude Code does not interpolate env vars in JSON — use the keyring populate script above to inject the token.

---

## Telemetry

By default, jcodemunch-mcp sends anonymous token-savings counts to `https://j.gravelle.us` (a community meter). Only `{"delta": <count>, "anon_id": "<uuid4>"}` is transmitted — no code, paths, or repo names.

Disabled in `~/.code-index/config.jsonc`:
```jsonc
{
  "share_savings": false
}
```

Or via env var (e.g. in `start.sh`):
```bash
export JCODEMUNCH_SHARE_SAVINGS=0
```

---

## Notes
- `JCODEMUNCH_HTTP_TOKEN` must be set — without it any local process can query the server
- `index_folder` requests serialize (sync behind `asyncio.to_thread`) — fine for multi-client use
- `JCODEMUNCH_RATE_LIMIT=N` available if per-IP throttling is needed
- `streamable-http` preferred over `sse` for multi-session local use
- Default endpoint: `http://127.0.0.1:8901`
