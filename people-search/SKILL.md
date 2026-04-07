---
name: people-search
metadata:
  version: 2.1.0
  tags: [people-search, b2b, enrichment, kol, recruiting, web-research]
description: >
  Search, qualify, and enrich people and companies. Use this skill whenever the
  user wants to find professionals, candidates, or KOLs by title, company,
  location, seniority, or audience; enrich known contacts with email, phone, or
  LinkedIn; research companies for industry, funding, tech stack, or hiring
  activity; look up someone's contact info; source candidates for recruiting;
  generate B2B lead lists; or perform background web research on people or
  organizations. Trigger this skill even when the user doesn't explicitly say
  "search" or "enrich" — any mention of finding contacts, sourcing, prospecting,
  looking up a person or company, or gathering business intelligence should
  activate it.
---

# Lessie — People Search & Enrichment

## Setup

Lessie supports two modes: **CLI** (default, recommended) and **MCP Server**.

### Mode A: CLI (default)

Install the Lessie CLI binary:

```bash
npm install -g @lessie/cli
```

Or use without installing:

```bash
npx @lessie/cli --version
```

First-time authorization:

```bash
lessie auth
```

This opens a browser for login/registration. Token is cached at `~/.lessie/oauth.json`.

Verify connection:

```bash
lessie status
```

### Mode B: MCP Server

Add to your MCP config (Claude Code `~/.claude/mcp.json`, Cursor `~/.cursor/mcp.json`, etc.):

```json
{
  "mcpServers": {
    "lessie": {
      "command": "npx",
      "args": ["-y", "@lessie/mcp-server"],
      "env": {
        "LESSIE_REMOTE_MCP_URL": "https://app.lessie.ai/mcp-server/mcp"
      }
    }
  }
}
```

### Uninstall

- **CLI:** `npm uninstall -g @lessie/cli && rm -rf ~/.lessie/`
- **MCP:** Remove the `"lessie"` entry from your `.mcp.json` and `rm -rf ~/.lessie/`

## Quick start

After setup, try saying to Claude:

- "Find Engineering Managers at Stripe in San Francisco"
- "Look up Sam Altman's contact info"
- "Research OpenAI — recent news and open job postings"

## Mode detection

Determine which mode to use at the start of each session:

1. Check if `lessie` CLI is available: run `lessie status`
2. If the command succeeds → use **CLI mode** (call tools via Bash)
3. If the command fails (not found) → attempt auto-install: `npm install -g @lessie/cli`
4. After install, run `lessie status` again to verify
5. If install succeeds → use **CLI mode**
6. If install fails (no npm, permission denied, network error, etc.) → check if MCP tools are available (`authorize`, `use_lessie`)
7. If MCP tools are available → use **MCP mode**
8. If neither → inform the user that installation failed and suggest manual install or MCP setup

## Credits & Pricing

Lessie is a credit-based service.

New accounts receive free trial credits. View your balance and purchase more at https://lessie.ai/pricing.

The agent will disambiguate company names before searching to avoid wasting credits on wrong results.

## Data & Privacy

- **Data sources:** Contact and company information is aggregated from publicly available sources (business directories, social profiles, corporate websites).
- **Query logging:** Search queries are logged for service improvement and abuse prevention. No query data is shared with third parties.
- **Data compliance:** Lessie follows applicable data protection regulations. Users are responsible for using retrieved contact data in compliance with local laws (GDPR, CAN-SPAM, etc.).
- **Privacy policy:** https://lessie.ai/privacy
- **Terms of service:** https://lessie.ai/terms-of-service

## Authorization

### CLI mode

1. Run `lessie status` to check token validity.
2. If `authorized: false` → run `lessie auth` to open browser for login.
3. After the user completes login, run `lessie status` again to confirm.

### MCP mode

1. Call `authorize` to check connection status.
2. **If already authorized** → proceed to use tools directly.
3. **If not authorized** → `authorize` returns an authorization URL. Tell the user you need to open a browser for Lessie login/registration, and open it using the appropriate system command:
   - macOS: `open "<url>"`
   - Linux: `xdg-open "<url>"`
   - Windows: `start "<url>"`
4. Tell the user the browser has been opened and they need to complete login/registration.
5. After the user confirms, call `authorize` again to verify the connection.
6. If authorization fails (timeout, denied, port conflict), follow the diagnostic hints returned by `authorize` and retry.

Always inform the user before opening the browser — never silently redirect.

## Agent behavior rules

### CRITICAL: Confirm before every credit-consuming action

Every Lessie tool call costs credits. Credit costs per tool:

| Tool | Cost |
|------|------|
| `find-people` | **20 credits** per search |
| `enrich-people` | 1 credit × number of people (only charged for successful matches) |
| `review-people` | 1 credit × number of people |
| `enrich-org` | 1 credit |
| `find-orgs` | 1 credit |
| `job-postings` | 1 credit |
| `company-news` | 1 credit |
| `web-search` | 1 credit |
| `web-fetch` | 1 credit |

**Before executing any command**, you MUST:

1. Tell the user what you are about to do and the estimated cost (e.g., "I'll enrich 3 people — this costs ~3 credits").
2. **Wait for explicit confirmation** before executing.
3. Never batch multiple credit-consuming calls without confirming the full plan first.

**Exception — skip confirmation** if the user has explicitly said they don't want to be prompted (e.g., "don't ask me every time", "just do it", "skip confirmations"). In that case, proceed directly but still log what you executed and the credits spent after each call.

### CRITICAL: Report credit usage after every call

After each conversation turn that involved one or more Lessie tool calls, append a one-line summary of credits consumed. Format:

> Used `<tool-name>`, cost <N> credit(s).

If multiple tools were called in the same turn, combine them:

> Used `web-search` + `enrich-org`, cost 2 credits total.

### CRITICAL: Read references before first CLI call

**Before executing any `lessie` CLI command for the first time in a session**, you MUST read [references/cli-reference.md](references/cli-reference.md) to learn the exact parameter syntax. Do NOT guess parameter names — the CLI uses `--filter` with JSON, not `--title`/`--company` style flags.

### Entity disambiguation

When a user mentions a company name that could refer to multiple entities (e.g., "Manus" could be Manus AI, Manus Bio, Manus Plus, etc.), disambiguate before searching:

1. **Ask the user** which company they mean, or present the top candidates and let them pick.
2. If context makes it unambiguous (e.g., user previously discussed AI agents), state your assumption and confirm: "你是指做 AI Agent 的 Manus AI (manus.im) 吗？"
3. **Never silently assume** one entity over another — wrong domain = wasted search credits and irrelevant results.

## Tools overview

### People

| Tool | CLI command | When to use |
|------|-------------|-------------|
| `find_people` | `lessie find-people` | Discover people by title, company, location, seniority, audience. Default strategy is `hybrid`. **If a request times out or fails, retry with `--strategy saas_only`** — it's faster (~30s vs ~60s) and more stable, though recall may be lower |
| `enrich_people` | `lessie enrich-people` | Enrich known people with full profiles. **Two paths**: B2B (via linkedin_url or name+domain → email, phone, work history) and KOL (via twitter/instagram/tiktok/youtube username → follower count, social links). Max 10 per call |
| `review_people` | `lessie review-people` | Deep-qualify **ambiguous** candidates via web research — skip for obvious matches/mismatches |

```bash
# Find people — uses --filter with JSON, NOT --title/--company flags
lessie find-people \
  --filter '{"person_titles":["Engineering Manager"],"organization_domains":["stripe.com"]}' \
  --checkpoint 'EMs at Stripe' \
  --strategy hybrid \
  --target-count 10

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

# Review people — deep-qualify from a previous search
lessie review-people \
  --search-id 'mcp_xxx' \
  --person-ids '["id1","id2"]' \
  --checkpoints '[{"key":"Relevance","description":"...","title":"Relevance","category":"career"}]'
```

### Companies

| Tool | CLI command | When to use |
|------|-------------|-------------|
| `find_organizations` | `lessie find-orgs` | Discover companies by name, keyword, location, size, funding |
| `enrich_organization` | `lessie enrich-org` | Get full profile for known company domain(s) — industry, employees, funding, tech stack |
| `get_company_job_postings` | `lessie job-postings` | View active job openings (needs `organization_id` from enrich) |
| `search_company_news` | `lessie company-news` | Find recent news articles (needs `organization_id` from enrich) |

```bash
# Find organizations
lessie find-orgs \
  --keyword-tags '["AI","SaaS"]' \
  --locations '["China"]' \
  --employees '["51,200"]'

# Enrich organization
lessie enrich-org --domains '["stripe.com"]'

# Job postings (needs org ID from enrich)
lessie job-postings --org-id '5f5e100...'

# Company news
lessie company-news --org-ids '["5f5e100..."]'
```

### Web research

| Tool | CLI command | When to use |
|------|-------------|-------------|
| `web_search` | `lessie web-search` | General web search; cached results make follow-up `web_fetch` free |
| `web_fetch` | `lessie web-fetch` | Extract specific info from a URL via AI summarization |

```bash
# Web search
lessie web-search --query 'OpenAI official website' --count 5

# Web fetch
lessie web-fetch --url 'https://example.com' --instruction 'Extract job title and company'
```

## Detailed references

- **CLI command examples & MCP calling**: See [references/cli-reference.md](references/cli-reference.md)
- **Workflow patterns** (domain resolution, company research, search+qualify): See [references/workflow-patterns.md](references/workflow-patterns.md)
- **Domain resolution decision tree**: See [references/domain-resolution.md](references/domain-resolution.md)

## Key constraints

- `enrich_people` / `enrich_organization`: max 10 per call; split larger lists into batches
- `find_people` / `find_organizations`: paginated — use `--page` for more results
- `web_search` caches page content; if a result has `has_content: true`, calling `web_fetch` on that URL is instant
- Seniority levels: `owner`, `founder`, `c_suite`, `partner`, `vp`, `head`, `director`, `manager`, `senior`, `entry`, `intern`
- For people enrichment, providing `domain` (company domain) alongside name greatly improves match accuracy
- CLI output is JSON on stdout, status messages on stderr — parse stdout for data
