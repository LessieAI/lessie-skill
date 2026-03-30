---
name: lessie
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
| `enrich_people` | `lessie enrich-people` | Fill missing profile data for known individuals (email, phone, LinkedIn, work history) |
| `review_people` | `lessie review-people` | Deep-qualify **ambiguous** candidates via web research — skip for obvious matches/mismatches |

### Companies

| Tool | CLI command | When to use |
|------|-------------|-------------|
| `find_organizations` | `lessie find-orgs` | Discover companies by name, keyword, location, size, funding |
| `enrich_organization` | `lessie enrich-org` | Get full profile for known company domain(s) — industry, employees, funding, tech stack |
| `get_company_job_postings` | `lessie job-postings` | View active job openings (needs `organization_id` from enrich) |
| `search_company_news` | `lessie company-news` | Find recent news articles (needs `organization_id` from enrich) |

### Web research

| Tool | CLI command | When to use |
|------|-------------|-------------|
| `web_search` | `lessie web-search` | General web search; cached results make follow-up `web_fetch` free |
| `web_fetch` | `lessie web-fetch` | Extract specific info from a URL via AI summarization |

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
