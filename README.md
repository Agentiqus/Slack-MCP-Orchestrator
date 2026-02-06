# Slack MCP Orchestrator

Connect AI tools (Cursor, Claude Desktop) to Slack with scoped, per-user permissions. Users choose what their AI can access, admins block sensitive channels org-wide, and the Slack bot token never leaves the server.

## What It Does

- **Users** open the App Home, toggle read/write permissions, and click "Generate MCP Config"
- **Admins** see an extra section to block specific channels org-wide
- **AI tools** connect via the generated config — every request is authenticated with a per-user JWT and permission-checked server-side

## Quick Setup (Slack CLI Sandbox)

### Prerequisites

- [Node.js 18+](https://nodejs.org/)
- [Slack CLI](https://api.slack.com/automation/cli/install) installed and authenticated (`slack login`)

### 1. Clone and install

```bash
git clone https://github.com/Agentiqus/Slack-MCP-Orchestrator.git
cd Slack-MCP-Orchestrator
npm install
```

### 2. Create the `.env` file

```bash
cp .env.sample .env
```

Generate a signing secret and add it:

```bash
echo "MCP_SIGNING_SECRET=$(openssl rand -hex 32)" >> .env
```

### 3. Start with Slack CLI

```bash
slack run
```

The CLI handles authentication, token injection, and manifest syncing. You'll see:

```
App running on port 3000 — Socket Mode (dev)
MCP API server listening on port 3001
```

### 4. Open the app in Slack

Go to your sandbox workspace, find **MCP Orchestrator** in the Apps section, and click the **Home** tab.

### 5. Set permissions and generate config

1. Click **Edit Permissions** — check the scopes you want (channels read/write, chat, users)
2. Click **Generate MCP Config** — copy the JSON from the modal

### 6. Connect your AI tool

Paste the config into `~/.cursor/mcp.json` (for Cursor) or your Claude Desktop config. Your AI tool now has scoped access to Slack.

## Project Structure

```
app.ts                          # Dual-mode startup (Socket Mode dev / HTTP prod)
src/
  db/
    database.ts                 # SQLite init, schema, WAL mode
    installation-store.ts       # Multi-tenant OAuth token storage
  auth/
    tokens.ts                   # JWT sign/verify/revoke/cleanup
  permissions/
    engine.ts                   # User permission CRUD + merge logic
    channels.ts                 # Admin channel blocklist (org-wide)
    admin.ts                    # Admin detection via Slack API
  api/
    server.ts                   # Express MCP API (rate-limited, auth middleware)
    middleware.ts               # JWT Bearer auth
    tools.ts                    # MCP tool definitions → Slack API mapping
  views/
    home-builder.ts             # Shared App Home builder (user + admin sections)
listeners/
  events/                       # app_home_opened, app_uninstalled
  actions/                      # edit permissions, generate config, block/unblock channels
  views/                        # modal submissions
packages/
  slack-mcp-client/             # Published npm package (@agentiqus/slack-mcp-client)
```

## MCP Tools Available

| Tool | Operation | What it does |
|---|---|---|
| `slack_list_channels` | read | List public channels |
| `slack_read_channel` | read | Read messages from a channel |
| `slack_post_message` | write | Post a message to a channel |
| `slack_reply_thread` | write | Reply to a thread |
| `slack_list_users` | read | List workspace members |

## Security

- Bot token **never leaves the server** — users only get a scoped JWT
- JWTs are signed with HMAC-SHA256, verified on every request
- One active token per user — generating a new config revokes the old one
- Permission changes auto-revoke existing tokens
- Admin channel blocklist enforced server-side on every API call
- Token cleanup runs on startup and every 24 hours
- Rate limiting: 60 requests/min per user

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `MCP_SIGNING_SECRET` | Yes | 256-bit secret for JWT signing (`openssl rand -hex 32`) |
| `MCP_API_URL` | Prod only | Public URL of the MCP API (e.g. `https://your-host.com/api/mcp`) |
| `SLACK_CLIENT_ID` | Prod only | From Slack app Basic Information |
| `SLACK_CLIENT_SECRET` | Prod only | From Slack app Basic Information |
| `SLACK_SIGNING_SECRET` | Prod only | From Slack app Basic Information |
| `SLACK_STATE_SECRET` | Prod only | Random secret for OAuth state (`openssl rand -hex 32`) |
| `DATA_DIR` | Optional | Override SQLite database directory (default: `./data`) |
| `PORT` | Optional | Server port (default: 3000) |
| `MCP_API_PORT` | Optional | MCP API port in Socket Mode (default: 3001) |

## Going to Production

To deploy this as a publicly installable Slack app, you need a Node.js host with persistent storage (Fly.io, Railway, Render, or any VPS). The app already supports multi-tenant OAuth — each workspace that installs it gets its own bot token stored in SQLite, and the `app.ts` entry point automatically switches from Socket Mode (local dev) to HTTP mode (production) based on whether `SLACK_APP_TOKEN` is set. Create a new Slack app at [api.slack.com/apps](https://api.slack.com/apps) in a non-sandbox workspace, configure the Event Subscriptions and Interactivity request URLs to point to `https://<YOUR_HOST>/slack/events`, set the OAuth redirect URL to `https://<YOUR_HOST>/slack/oauth_redirect`, add your app credentials as environment variables on your host, and enable public distribution under Manage Distribution. The install URL at `https://<YOUR_HOST>/slack/install` will handle the full OAuth flow for any workspace. For Slack Marketplace submission, you'll additionally need a landing page, support page, privacy policy, app icon, screenshots, and at least 5 active workspace installs.

## License

MIT
