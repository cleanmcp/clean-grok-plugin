# Clean

[Clean](https://tryclean.ai) finds and researches B2B leads that match your ideal customer profile.
This repo holds Clean's listing manifests for every agent marketplace (Grok Bot, Cursor, Claude, ChatGPT/Codex, the MCP Registry and more), so your agent can run that research and work with the results.

The plugin contains no server code. It only points at `https://app.tryclean.ai/v1/mcp`, which Clean hosts.

Layout: `plugin.json` at the repo root is the portable [Agent Plugins](https://agent-plugins.org) manifest, `.cursor-plugin/plugin.json` is the Cursor Plugin manifest (adds the logo and marketplace metadata), `.claude-plugin/` is the Claude Code plugin + marketplace, `.agents/plugins/marketplace.json` is the Codex marketplace, `openai/` is the ChatGPT plugin package, `server.json` is the MCP Registry entry and `mcp.json` holds the MCP server definition. The MCP server name is `clean`.

## Marketplaces

| Marketplace | Status | Listing / proof | Date | Needs owner |
| --- | --- | --- | --- | --- |
| Grok Bot (x.ai) | Submitted | x.ai plugin form, HTTP 200 `{"success":true}` | 2026-11-04 | Wait for x.ai review |
| Cursor Marketplace | Submitted | Email thread with Sam at Cursor; deeplink below works today | 2026-11-04 | Reply to Cursor thread |
| MCP Registry (modelcontextprotocol.io) | Ready, publishes on merge | `server.json` + `.github/workflows/publish-mcp-registry.yml`; check `https://registry.modelcontextprotocol.io/v0.1/servers?search=io.github.cleanmcp/clean` | 2026-11-04 | Merge to main (Actions OIDC must be enabled) |
| Claude connector directory | Blocked on owner | https://claude.ai/directory/manage → Submit new → MCP connector | 2026-11-04 | Paid Claude plan login as hello@, reviewer test account |
| Claude Code plugin | Live on merge | `/plugin install clean --marketplace cleanmcp/clean-grok-plugin` | 2026-11-04 | Merge to main |
| Anthropic plugin directory | Blocked on owner | https://claude.com/partners/mcp → Submit a plugin | 2026-11-04 | Same login as connector directory |
| ChatGPT plugin directory | Package ready | `openai/` folder, ZIP via `make-openai-zip` below; upload at https://platform.openai.com/plugins | 2026-11-04 | OpenAI org login, developer identity verification, domain challenge at `app.tryclean.ai/.well-known/openai-apps-challenge`, reviewer test account |
| Codex marketplace | Live on merge | `codex plugin marketplace add cleanmcp/clean-grok-plugin` | 2026-11-04 | Merge to main |
| awesome-remote-mcp-servers | Row ready | https://github.com/punkpeye/awesome-remote-mcp-servers (table row below) | 2026-11-04 | GitHub fork + PR from an account with a token |
| mcpservers.org | see commit log | https://mcpservers.org/submit (free) | 2026-11-04 | - |
| PulseMCP | see commit log | https://www.pulsemcp.com/submit | 2026-11-04 | - |
| mcp.so | Blocked on owner | https://mcp.so/submit (free tier; $39 optional) | 2026-11-04 | GitHub sign-in |
| Glama | Blocked on owner | https://glama.ai/mcp/servers/add | 2026-11-04 | Glama sign-in |
| Smithery | Not possible | Self-serve submissions shut down 2026-10-31; curated by Arcade | 2026-11-04 | - |
| Zapier MCP / Make | Not applicable | They expose their own tools as MCP servers; no third-party remote MCP directory | 2026-11-04 | - |

awesome-remote-mcp-servers row: `| Clean | Sales | \`https://app.tryclean.ai/v1/mcp\` | OAuth2.1 & API Key | [Clean](https://tryclean.ai) |`

### Build the ChatGPT ZIP

```bash
cd openai && zip -r ../clean-openai-plugin.zip . -x '.DS_Store'
```

## Install

**Grok Bot:** open **Plugins** (marketplace), search for **Clean**, click **Install**, then sign in with Clean when the browser opens.

**Cursor one-click:** [Add Clean to Cursor](cursor://anysphere.cursor-deeplink/mcp/install?name=clean&config=eyJ1cmwiOiJodHRwczovL2FwcC50cnljbGVhbi5haS92MS9tY3AifQ==) (`cursor://anysphere.cursor-deeplink/mcp/install?name=clean&config=<base64 of {"url":"https://app.tryclean.ai/v1/mcp"}>`).

**Claude Code:** `/plugin install clean --marketplace cleanmcp/clean-grok-plugin`, then sign in with Clean when the browser opens.

**Claude.ai / Claude Desktop:** Settings → Connectors → Add custom connector → URL `https://app.tryclean.ai/v1/mcp`.

**ChatGPT / Codex:** `codex plugin marketplace add cleanmcp/clean-grok-plugin`, or in ChatGPT Settings → Connectors → Create → URL `https://app.tryclean.ai/v1/mcp`. Agents SDK: `HostedMCPTool(tool_config={"type": "mcp", "server_label": "clean", "server_url": "https://app.tryclean.ai/v1/mcp", "require_approval": "never"})`.

**VS Code / Copilot:** `code --add-mcp '{"name":"clean","type":"http","url":"https://app.tryclean.ai/v1/mcp"}'`.

**Cursor:** open **Customize → Plugins**, search for **Clean** in the Cursor Marketplace, click **Install** and pick user or project scope, then sign in with Clean when asked. If Cursor cannot finish the Clean sign-in, use the API key setup below.

**Install from this repo (before the listing is live):** copy this folder to `~/.cursor/plugins/local/clean`, run **Developer: Reload Window**, and confirm the `clean` MCP server shows up under Customize.

**Any other MCP client:** add a remote (Streamable HTTP) MCP server:

```json
{
  "mcpServers": {
    "clean": {
      "type": "http",
      "url": "https://app.tryclean.ai/v1/mcp"
    }
  }
}
```

You need a Clean account. Start at [tryclean.ai](https://tryclean.ai).

## Sign in

Auth is OAuth 2.1 ("Sign in with Clean"): on first use the client opens a browser, you sign in to Clean, pick a workspace and allow access (read-only or full). There is no key to paste and nothing is stored in this repo.

If your client cannot do OAuth, create a workspace API key in Clean under **Settings → API & MCP** and send it as a header. Keep the key in an environment variable, never in a file you commit:

```json
{
  "mcpServers": {
    "clean": {
      "url": "https://app.tryclean.ai/v1/mcp",
      "headers": { "Authorization": "Bearer ${env:CLEAN_API_KEY}" }
    }
  }
}
```

Revoke a connected app or key at any time in **Settings → API & MCP**.

## What the agent can do

| Tool | What it does |
| --- | --- |
| `research`, `reply`, `get_run`, `cancel_run` | Ask Clean's agent to find and research leads, answer its questions, follow progress, stop a run |
| `get_leads`, `list_conversations` | Read the qualified leads of a research run and earlier results |
| `list_icps`, `add_context` | Read your ideal customer profiles and give Clean background to use |
| `connections` | Search your own synced LinkedIn 1st-degree connections |
| `leads_read`, `leads_write` | Read saved leads and organize them into lists |
| `sequences_read`, `sequences_write` | Read sequences and draft them; drafting never sends a message |

The hosted server is the source of truth for tool names and schemas. To check the connection, ask your agent to "list my Clean ICPs".

## Support

hello@tryclean.ai

## License

[MIT](LICENSE)
