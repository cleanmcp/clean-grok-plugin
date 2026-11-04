---
name: get-started
description: Set up Clean and run the first lead research from this chat.
---

# Get started with Clean

Clean finds and researches B2B leads that match an ideal customer profile (ICP).

1. If the Clean MCP server is not connected yet, ask the user to sign in with Clean when prompted. No API key is needed.
2. Call `list_icps`. If the workspace has no ICP, ask the user to describe their ideal customer in one sentence (industry, company size, role) and pass it to `research`.
3. Call `research` with the ICP, then poll `get_run` until it finishes. Runs take minutes; tell the user and continue when done.
4. Call `get_leads` and show the qualified leads with the reason each one matched.
5. Offer next steps: `connections` to find warm paths, `sequences_write` to draft outreach. Drafting never sends a message; only the user sends from Clean.

Stay inside the signed-in workspace. Never invent leads, contact details or reasons that the tools did not return.
