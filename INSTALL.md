# Installing the PrivCo MCP server

This skill teaches Claude how to use the `privco-data-mcp` MCP server. To use
it, the server must be installed and registered with your MCP client (Claude
Code, Claude Desktop, claude.ai, etc.).

You will need a **PrivCo API key**. Contact support@privco.com if you don't
have one.

## 1. Install the server (Node.js ≥ 18 required)

The server is published to npm: <https://www.npmjs.com/package/privco-data-mcp>

**Option A — Global install (recommended, simplest)**

```bash
npm install -g privco-data-mcp
```

This puts a `privco-data-mcp` binary on your `PATH`.

**Option B — Run on-demand via npx (no install needed)**

```bash
npx -y privco-data-mcp@latest
```

Use this command form in your MCP client config if you prefer not to install
globally.

## 2. Register the server with your MCP client

The server reads its API key from the `PRIVCO_API_KEY` environment variable.
Set it in the `env` block of your MCP client config — never hardcode it in
source files or commit it to git.

### Claude Code (CLI)

Edit `~/.claude.json` (or the project-local `.mcp.json`) and add an entry
under `mcpServers`:

```json
{
  "mcpServers": {
    "privco-data-mcp": {
      "command": "privco-data-mcp",
      "args": [],
      "env": {
        "PRIVCO_API_KEY": "your_api_key_here"
      }
    }
  }
}
```

Or, using `npx` (no global install):

```json
{
  "mcpServers": {
    "privco-data-mcp": {
      "command": "npx",
      "args": ["-y", "privco-data-mcp@latest"],
      "env": {
        "PRIVCO_API_KEY": "your_api_key_here"
      }
    }
  }
}
```

You can also register via the CLI:

```bash
claude mcp add privco-data-mcp \
  --env PRIVCO_API_KEY=your_api_key_here \
  -- privco-data-mcp
```

After adding the server, restart Claude Code. Verify the tools loaded:

```bash
claude mcp list
```

You should see `privco-data-mcp` listed. Inside a session, the tools appear as
`mcp__privco-data-mcp__match`, `mcp__privco-data-mcp__company_search`, etc.

### Claude Desktop

Edit `claude_desktop_config.json`:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Use the same `mcpServers` block shown above, then fully quit and relaunch
Claude Desktop.

### Other MCP clients

The PrivCo server speaks the standard MCP **stdio** transport. Any
spec-compliant client can run it via:

- `command: "privco-data-mcp"` (if globally installed), or
- `command: "npx"`, `args: ["-y", "privco-data-mcp@latest"]`

Make sure `PRIVCO_API_KEY` is in the spawned process's environment.

## 3. Confirm it works

Ask Claude something like:

> "Look up Stripe in PrivCo and give me the latest valuation."

If the tools are wired up, Claude will call `mcp__privco-data-mcp__match`,
then `mcp__privco-data-mcp__profile` / `mcp__privco-data-mcp__vc_deals`, and
return the answer. If you get a 401/403, the API key isn't being passed
through — re-check the `env` block.

## 4. Troubleshooting

| Symptom | Likely cause | Fix |
|--------|--------------|-----|
| Tools don't appear after restart | Config JSON malformed | Validate with `jq . < ~/.claude.json` |
| 401 / 403 on every call | API key missing or wrong | Verify `PRIVCO_API_KEY` is set in the server's `env` block; restart client |
| `command not found: privco-data-mcp` | Global npm bin not on PATH | Use the `npx` form instead, or add the npm global bin dir to `PATH` |
| `Cannot find module` errors | Node version too old | Upgrade to Node.js ≥ 18 |
| Tools time out | Network egress blocked | The server connects to PrivCo's API over HTTPS — confirm outbound HTTPS is allowed |
| Need a different base URL | Want to point at a non-default PrivCo endpoint | Set `PRIVCO_API_BASE_URL` in the same `env` block alongside `PRIVCO_API_KEY` (e.g. `"PRIVCO_API_BASE_URL": "https://api.privco.com/v3"`). Optional; uses the default PrivCo API endpoint when unset |

## Security notes

- Never commit `PRIVCO_API_KEY` to git. Always put it in your MCP client
  config (which is local to your machine) or in an OS-level secret store.
- The MCP server is a thin wrapper that forwards your key to PrivCo's API. It
  does not log or persist requests.
- If sharing this skill with others, **do not** include your API key. Each
  user supplies their own.
