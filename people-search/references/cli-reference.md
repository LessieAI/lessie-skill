# CLI Tool Calling Reference

## Command examples

```bash
# Search people
lessie find-people \
  --filter '{"person_titles":["CTO"],"person_locations":["US"]}' \
  --checkpoint 'CTOs at AI startups in the US' \
  --strategy hybrid \
  --target-count 20

# Enrich people (B2B) — linkedin_url is best; fallback: name + domain
lessie enrich-people \
  --people '[{"linkedin_url":"https://www.linkedin.com/in/samaltman/"}]'

# Enrich people (B2B) — name + domain fallback
lessie enrich-people \
  --people '[{"first_name":"Sam","last_name":"Altman","domain":"openai.com"}]'

# Enrich people (B2B) — include personal emails
lessie enrich-people \
  --people '[{"first_name":"Sam","last_name":"Altman","domain":"openai.com"}]' \
  --include-personal-emails

# Enrich people (KOL) — Twitter/X
lessie enrich-people \
  --people '[{"twitter_screen_name":"elonmusk"}]'

# Enrich people (KOL) — Instagram
lessie enrich-people \
  --people '[{"instagram_username":"natgeo"}]'

# Enrich people (KOL) — TikTok
lessie enrich-people \
  --people '[{"tiktok_username":"charlidamelio"}]'

# Enrich people (KOL) — YouTube
lessie enrich-people \
  --people '[{"youtube_username":"MrBeast"}]'

# Review people (from a previous search)
lessie review-people \
  --search-id 'mcp_xxx' \
  --person-ids '["id1","id2"]' \
  --checkpoints '[{"key":"Relevance","description":"...","title":"Relevance","category":"career"}]'

# Find organizations
lessie find-orgs \
  --keyword-tags '["AI","SaaS"]' \
  --locations '["China"]' \
  --employees '["51,200"]'

# Enrich organization
lessie enrich-org --domains '["stripe.com"]'

# Job postings (needs org ID from enrich/search)
lessie job-postings --org-id '5f5e100...'

# Company news
lessie company-news --org-ids '["5f5e100..."]'

# Web search
lessie web-search --query 'OpenAI official website' --count 5

# Web fetch
lessie web-fetch --url 'https://example.com' --instruction 'Extract job title and company'
```

Output is always JSON on stdout. Use `--pretty` for human-readable formatting.

## MCP mode — call via MCP tools

Use `use_lessie(tool="tool_name", arguments={...})` or call remote tools directly if already exposed.

Parameter names are identical to the CLI arguments.

## Generic tool call

`tools` and `call` are the universal fallback for any tool — including shortcuts that fail due to argument changes or server-side updates.

```bash
lessie tools                                    # List all available remote tools with full parameter schemas
lessie call <tool_name> --args '{"key":"val"}'  # Call any tool by name with raw JSON arguments
```

**`lessie tools`** fetches the latest tool list directly from the remote server. Each tool entry includes:
- Tool name (the identifier you pass to `lessie call`)
- Description
- Full input schema (`properties`, `required` fields, types, defaults)

**`lessie call`** bypasses the CLI shortcut commands and sends the arguments directly to the remote tool. This is useful when:
- A shortcut command doesn't exist for a tool
- The remote server has added new tools not yet covered by CLI shortcuts
- You need to pass parameters that the shortcut doesn't expose

### Error recovery: fall back to `tools` + `call`

**When a CLI shortcut command fails 3+ times** (argument errors, schema mismatches, unexpected parameters), stop retrying the shortcut and switch to the generic `tools` + `call` workflow:

1. Run `lessie tools` to fetch the **current** remote tool schema (the server may have updated parameter names or added required fields since the CLI was built).
2. Find the matching tool name (e.g., `find_people` for `lessie find-people`, `web_fetch` for `lessie web-fetch`).
3. Read its `inputSchema` to understand the exact parameter names, types, and required fields.
4. Call it via `lessie call <tool_name> --args '{ ... }'` using the correct schema.

Example:
```bash
# Step 1: Shortcut fails repeatedly
lessie web-fetch --url 'https://...' --instruction '...'
# Error: --url is required. URL to fetch   (3rd failure)

# Step 2: Fall back to tools + call
lessie tools                                    # Check current schema for web_fetch
lessie call web_fetch --args '{"url":"https://...","instruction":"..."}'
```
