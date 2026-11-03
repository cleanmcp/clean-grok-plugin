# Security

## What this repo contains

Only a plugin manifest (`.cursor-plugin/plugin.json`), an MCP server address (`mcp.json`), a logo and docs. No code runs on your machine, and no credentials are stored here.

## How access works

- The plugin connects to `https://app.tryclean.ai/v1/mcp` over HTTPS.
- Sign-in is OAuth 2.1 with PKCE. Each grant belongs to one person and one workspace and is read-only or full.
- Every call runs Clean's normal membership and role checks. Tools that change data act only inside your workspace; drafting a sequence never sends a message.
- Revoke access any time in Clean under **Settings → API & MCP**.

## Reporting a vulnerability

Email hello@tryclean.ai with the details and steps to reproduce. Please do not open a public issue.
