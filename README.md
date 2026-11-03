# Clean

[Clean](https://tryclean.ai) finds and researches B2B leads that match your ideal customer profile.
This plugin connects Grok Bot and Cursor to Clean's hosted MCP server, so your agent can run that research and work with the results.

The plugin contains no server code. It only points at `https://app.tryclean.ai/v1/mcp`, which Clean hosts.

## Install

**Grok Bot or Cursor:** open the plugin marketplace, search for **Clean**, click **Install**, then sign in when asked. If Cursor cannot finish the Clean sign-in, use the API key setup below.

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
