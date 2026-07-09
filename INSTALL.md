# Accessing the PrivCo MCP server

This skill teaches Claude how to use the PrivCo MCP tools. To use them, your
MCP client must be connected to the PrivCo MCP server — and there are **two
ways to connect**:

| | **Remote connector** (recommended) | **Local server** (npm) |
|---|---|---|
| What it is | PrivCo-hosted MCP endpoint at `https://mcp.privco.com/mcp` | The `privco-data-mcp` npm package running on your machine |
| Auth | OAuth — sign in with your PrivCo account in the browser | `PRIVCO_API_KEY` environment variable |
| Works in | claude.ai (web), Claude Desktop, Claude Code, Codex CLI, ChatGPT, Microsoft 365 Copilot (Copilot Studio), any Streamable-HTTP + OAuth 2.1 MCP client | Claude Code, Claude Desktop, Codex CLI, GitHub Copilot CLI, Antigravity, any stdio MCP client |
| Requires | A PrivCo account with API access enabled | A PrivCo API key + Node.js ≥ 18 |
| Best for | Most users — nothing to install, no key to manage | Automation, CI, air-gapped configs, or when you prefer key-based auth |

Both expose the **same 17 tools** with identical names and schemas.

- **To get API access turned on, a plan, or an API key** → contact <sales@privco.com>.
- **For technical setup help or troubleshooting** → contact <support@privco.com>.

> **Which path does my client use?** claude.ai / ChatGPT / Microsoft 365
> Copilot → **Remote**. Claude Code / Codex CLI → **either** (both shown
> below). GitHub Copilot CLI / Antigravity / other stdio tools → **Local**.

---

## Easiest path for Claude Code — the PrivCo plugin

If you use **Claude Code**, the official PrivCo plugin bundles all 17 MCP
tools **and** this skill in one install, and prompts for your API key (stored
securely in your system keychain — never written to a config file):

```
/plugin marketplace add privco/privco-claude-plugin
/plugin install privco-data@privco
```

Enable the plugin, paste your key when prompted, and you're done — no config
editing. (Need a key? Contact <sales@privco.com>.) Requires Node.js ≥ 18.
Source: <https://github.com/privco/privco-claude-plugin>. Prefer to wire
things manually, or using a different client? Use the two options below.

---

## Option 1 — Remote connector (no install)

The PrivCo MCP server is hosted at:

```
https://mcp.privco.com/mcp
```

Authentication is OAuth 2.1: your client opens a browser window, you sign in
with your PrivCo account and approve access, and tokens are managed for you.
No API key ever touches your config files.

> **Plan note:** custom connectors on claude.ai require a paid Claude plan,
> and ChatGPT's developer mode requires a paid ChatGPT plan. Claude Code and
> Codex CLI work on any plan with MCP support.

### claude.ai (web) and Claude Desktop

1. Go to **Settings → Connectors → Add custom connector**.
2. Fill in:
   - **Name**: `PrivCo` (any name works)
   - **Remote MCP server URL**: `https://mcp.privco.com/mcp`
   - **OAuth Client ID / Client Secret**: leave **blank** (the server
     supports dynamic client registration)
3. Click **Add**, then **Connect** — a PrivCo sign-in page opens. Sign in
   and approve.
4. In a chat, enable the connector via the tools/connectors menu. The 17
   PrivCo tools are now available.

### Claude Code (CLI)

```bash
claude mcp add --transport http privco-data-mcp https://mcp.privco.com/mcp
```

Then inside a session run `/mcp` and pick `privco-data-mcp` to complete the
browser sign-in. Tools appear as `mcp__privco-data-mcp__match`,
`mcp__privco-data-mcp__company_search`, etc.

### Codex CLI (OpenAI)

Codex CLI speaks the remote Streamable-HTTP transport with OAuth:

```bash
codex mcp add privco --url https://mcp.privco.com/mcp
codex mcp login privco
```

`codex mcp login` runs the browser sign-in. After it completes, the PrivCo
tools are available in a Codex session. (Prefer key-based auth? Use the local
form in Option 2 instead.)

### ChatGPT

ChatGPT currently requires **developer mode** for custom MCP connectors:

1. **Settings → Apps & Connectors → Advanced settings** — enable
   **Developer mode**.
2. **Create** a new app/connector with:
   - **MCP server URL**: `https://mcp.privco.com/mcp`
   - **Authentication**: OAuth
3. Complete the PrivCo sign-in when prompted, then enable the app in a chat.

### Microsoft 365 Copilot (via Copilot Studio)

This is set up by an **M365 administrator**, not per-user: in **Copilot
Studio**, add PrivCo as a custom (MCP) tool at `https://mcp.privco.com/mcp`
with **API-key** authentication (header `x-api-key`) — Copilot Studio's manual
connector uses an API key here rather than the browser OAuth used by the other
remote clients — build a "PrivCo" agent, and
publish it to your team. PrivCo provides admins a dedicated walkthrough — ask
your PrivCo contact or <support@privco.com>.

### Other MCP clients

Any client that speaks MCP **Streamable HTTP** with OAuth 2.1 (dynamic
client registration + PKCE) can connect to `https://mcp.privco.com/mcp`.

---

## Option 2 — Local server via npm (stdio)

Use this when you prefer API-key auth, need to run without a browser
sign-in (automation/CI), or want the server on your own machine.

You will need a **PrivCo API key** — contact <sales@privco.com> to obtain one.

### 1. Install the server (Node.js ≥ 18 required)

The server is published to npm: <https://www.npmjs.com/package/privco-data-mcp>

**Option A — Global install (simplest)**

```bash
npm install -g privco-data-mcp
```

This puts a `privco-data-mcp` binary on your `PATH`.

**Option B — Run on-demand via npx (no install needed)**

```bash
npx -y privco-data-mcp@latest
```

Use this command form in your MCP client config if you prefer not to install
globally. Always pin `@latest` so you point at the current production API.

### 2. Register the server with your MCP client

The server reads its API key from the `PRIVCO_API_KEY` environment variable.
Set it in the `env` block of your MCP client config — never hardcode it in
source files or commit it to git.

#### Claude Code (CLI)

Edit `~/.claude.json` (or the project-local `.mcp.json`) and add an entry
under `mcpServers`:

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

Or register via the CLI:

```bash
claude mcp add privco-data-mcp \
  --env PRIVCO_API_KEY=your_api_key_here \
  -- npx -y privco-data-mcp@latest
```

After adding the server, restart Claude Code. Verify with `claude mcp list`
— you should see `privco-data-mcp`. Tools appear as
`mcp__privco-data-mcp__match`, `mcp__privco-data-mcp__company_search`, etc.

#### Codex CLI (OpenAI)

```bash
codex mcp add privco \
  --env PRIVCO_API_KEY=your_api_key_here \
  -- npx -y privco-data-mcp@latest
```

Verify with `codex mcp list`. (Prefer browser sign-in? Use the remote form
in Option 1 instead.)

#### Claude Desktop

Edit `claude_desktop_config.json`:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Use the same `mcpServers` block shown above, then fully quit and relaunch
Claude Desktop.

(For Claude Desktop, the remote connector in Option 1 is usually the easier
path — no config-file editing.)

#### GitHub Copilot CLI, Antigravity, and other stdio clients

The PrivCo server speaks the standard MCP **stdio** transport. Any
spec-compliant client can run it via:

- `command: "privco-data-mcp"` (if globally installed), or
- `command: "npx"`, `args: ["-y", "privco-data-mcp@latest"]`

Register it in the client's MCP config (each client has its own config file —
e.g. GitHub Copilot CLI, Antigravity `mcp_config.json`, Cursor `.cursor/mcp.json`),
and make sure `PRIVCO_API_KEY` is in the spawned process's environment.

---

## Confirm it works

Ask Claude something like:

> "Look up Stripe in PrivCo and give me the latest valuation."

If the tools are wired up, Claude will call `match`, then `profile` /
`vc_deals`, and return the answer.

> **Tool-name note:** the 17 tools are identical on both transports. The
> full tool id your client shows is prefixed with the server/connector name
> you registered — e.g. `mcp__privco-data-mcp__match` in Claude Code, or
> under whatever name you gave the connector in claude.ai.

## A note on Gemini

As of mid-2026, Google has **retired the personal-account Gemini CLI**; its
replacement, **Antigravity** (`agy`), is a standard stdio MCP client — use the
local form above. Consumer Gemini (web/app) supports only curated connectors;
custom MCP servers require **Gemini Enterprise**. (AI-vendor MCP support moves
fast — if any of this has changed, email <support@privco.com>.)

## Troubleshooting

| Symptom | Likely cause | Fix |
|--------|--------------|-----|
| **Remote:** sign-in loop or "access denied" after login | Your PrivCo account doesn't have API access enabled | Contact <sales@privco.com> to enable API access on your account |
| **Remote:** connector added but tools missing in chat | Connector not enabled for the conversation | Enable it in the chat's tools/connectors menu; reconnect if it was added long ago |
| **Remote (ChatGPT):** can't add the connector | Developer mode not enabled | Settings → Apps & Connectors → Advanced settings → Developer mode |
| **Local:** tools don't appear after restart | Config JSON malformed | Validate with `jq . < ~/.claude.json` |
| **Local:** 401 / 403 on every call | API key missing or wrong | Verify `PRIVCO_API_KEY` is set in the server's `env` block; restart client |
| **Local:** `command not found: privco-data-mcp` | Global npm bin not on PATH | Use the `npx` form instead, or add the npm global bin dir to `PATH` |
| **Local:** `Cannot find module` errors | Node version too old | Upgrade to Node.js ≥ 18 |
| Tools time out | Network egress blocked | The server connects to PrivCo's API over HTTPS — confirm outbound HTTPS is allowed |
| **Local:** need a different base URL | Want to point at a non-default PrivCo endpoint | Set `PRIVCO_API_BASE_URL` in the same `env` block alongside `PRIVCO_API_KEY`, using the endpoint URL PrivCo support provides. Optional; uses the default PrivCo API endpoint when unset |

Still stuck? Email <support@privco.com>.

## Security notes

- **Remote connector:** auth is OAuth — you never handle an API key. Revoke
  access anytime by removing the connector from your client; email
  <support@privco.com> to revoke server-side. Usage is subject to your
  PrivCo account terms and normal API usage metering.
- **Local server:** never commit `PRIVCO_API_KEY` to git. Always put it in
  your MCP client config (which is local to your machine) or in an OS-level
  secret store. The npm package is a thin wrapper over PrivCo's API — it
  does not itself log or persist your queries.
- If sharing this skill with others, **do not** include your API key. Each
  user supplies their own credentials.
