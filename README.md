# Lessie

> Finding the right person used to take hours of LinkedIn stalking, cross-referencing, and cold emails to dead inboxes. Now it takes one sentence.

[Lessie](https://lessie.ai) is an AI-native people search and enrichment platform. This repo is the **Lessie skill** — it turns [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Codex](https://github.com/openai/codex), or any agent that supports the [SKILL.md standard](https://github.com/anthropics/claude-code) into a recruiting sourcer, sales prospector, and business researcher.

**What it does:**

- **Find people** — by title, company, location, seniority, or audience
- **Enrich contacts** — email, phone, LinkedIn, work history for known individuals
- **Research companies** — industry, funding, tech stack, job postings, recent news
- **Qualify candidates** — deep web research to verify ambiguous matches
- **Web research** — general search and AI-powered page summarization

No API keys to configure. No SDKs to learn. Just talk to your agent.

## Get Started

1. **Create a free account** at [app.lessie.ai](https://app.lessie.ai/login?utm_source=github&utm_channel=skill) — new accounts include trial credits
2. **Install the skill** (see below)
3. **Run `/lessie`** in your agent — it will handle auth automatically

## Install — 30 seconds

**Requirements:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [Codex CLI](https://github.com/openai/codex), [Node.js](https://nodejs.org/) 18+

```bash
npx skills add LessieAI/lessie-skill -y -g
```

Restart your agent. The `/lessie` skill is now available.

## See it work

```
You:    /lessie

Claude: Let me check if Lessie CLI is available.

        Bash(lessie status 2>&1)
        ⎿  Error: Exit code 127
           (eval):1: command not found: lessie

        CLI not found. Let me install it.

        Bash(npm install -g @lessie/cli 2>&1)
        ⎿  added 2 packages in 3s

        Bash(lessie status 2>&1)
        ⎿  {
             "remote_url": "https://app.lessie.ai/mcp-server/mcp",
             "authorized": true,
           … +2 lines

        Lessie CLI installed and authorized. You're ready to go. For example:

        - Find people — "Find Engineering Managers at Stripe"
        - Look up contacts — "Get Sam Altman's contact info"
        - Research companies — "What's OpenAI been up to lately?"

        What would you like to search?

You:    Find Engineering Managers at Stripe

Claude: Bash(lessie find-people \
        --filter '{"person_titles":["Engineering Manager"],"organization_domains":["stripe.com"]}' \…)
        ⎿  {"success":true,"search_id":"mcp_5173a4e1a389","people":[…]}
           … +85 lines

        Found 10 Engineering Managers at Stripe:

        ┌─────┬──────────────────┬───────────────────────────────────────────┬───────────────────┬───────┬───────┐
        │  #  │ Name             │ LinkedIn                                  │ Industry          │ Email │ Phone │
        ├─────┼──────────────────┼───────────────────────────────────────────┼───────────────────┼───────┼───────┤
        │ 1   │ Andrew Wang      │ linkedin.com/in/andrew-y-wang             │ Internet          │ ✅    │ ✅    │
        │ 2   │ Shireesh Asthana │ linkedin.com/in/shireesh-asthana-a3b1b321 │ Internet          │ ✅    │ ✅    │
        │ 3   │ Erik Barbara     │ linkedin.com/in/erikbarbara               │ Network Security  │ ✅    │ ✅    │
        │ 4   │ Guy Alster       │ linkedin.com/in/guyalster                 │ Computer Software │ ✅    │ ✅    │
        │ 5   │ Tomer Elmalem    │ linkedin.com/in/tomer-elmalem             │ Computer Software │ ✅    │ ✅    │
        │ 6   │ George Bland     │ linkedin.com/in/georgebland               │ Computer Software │ ✅    │ ✅    │
        │ 7   │ Jona Okoth       │ linkedin.com/in/jona-okoth-5ba42635       │ Internet          │ ✅    │ ✅    │
        │ 8   │ Darren Jennings  │ linkedin.com/in/darren-jennings           │ Internet          │ ✅    │ ✅    │
        │ 9   │ Eoin Nugent      │ linkedin.com/in/eoinnugent                │ Internet          │ ✅    │ ✅    │
        │ 10  │ Clément Miao     │ linkedin.com/in/clementmiao               │ Computer Hardware │ ✅    │ ✅    │
        └─────┴──────────────────┴───────────────────────────────────────────┴───────────────────┴───────┴───────┘

        All emails verified. What would you like to do next?

        - Get contact details (email/phone) — using enrich-people
        - Filter by region — re-search with location filters
        - See more results — paginate to the next batch
```

### Alternative: MCP Server (no CLI needed)

Skip the CLI install. Add Lessie as an MCP server instead:

| Client | Config file |
|--------|-------------|
| Claude Code | `~/.claude/mcp.json` |
| Cursor | `~/.cursor/mcp.json` |
| Codex | `~/.codex/config.json` |

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

- **CLI + Skill:** `npm uninstall -g @lessie/cli && rm -rf ~/.lessie/ ~/.claude/skills/lessie`
- **MCP:** Remove the `"lessie"` entry from your MCP config and `rm -rf ~/.lessie/`
- **Codex:** `rm -rf ~/.codex/skills/lessie` (or `.agents/skills/lessie`)

## Tools

### People

| Tool | CLI Command | What it does |
|------|-------------|--------------|
| `find_people` | `lessie find-people` | Discover people by title, company, location, seniority. Default strategy: `hybrid`. If timeout, retry with `--strategy saas_only` (faster, lower recall) |
| `enrich_people` | `lessie enrich-people` | Fill missing profile data — email, phone, LinkedIn, work history. Max 10 per call |
| `review_people` | `lessie review-people` | Deep-qualify ambiguous candidates via web research. Skip for obvious matches |

### Companies

| Tool | CLI Command | What it does |
|------|-------------|--------------|
| `find_organizations` | `lessie find-orgs` | Discover companies by name, keyword, location, size, funding |
| `enrich_organization` | `lessie enrich-org` | Full company profile — industry, employees, funding, tech stack. Max 10 per call |
| `get_company_job_postings` | `lessie job-postings` | Active job openings (needs `organization_id` from enrich) |
| `search_company_news` | `lessie company-news` | Recent news articles (needs `organization_id` from enrich) |

### Web Research

| Tool | CLI Command | What it does |
|------|-------------|--------------|
| `web_search` | `lessie web-search` | General web search. Results are cached — follow-up `web_fetch` on cached URLs is free |
| `web_fetch` | `lessie web-fetch` | AI-powered page summarization from any URL |

## Credits & Pricing

Lessie is credit-based. New accounts get free trial credits. View balance and buy more at [lessie.ai/pricing](https://lessie.ai/pricing).

The agent disambiguates company names before searching to avoid wasting credits on wrong results.

## References

| Doc | What it covers |
|-----|---------------|
| [CLI Reference](references/cli-reference.md) | Command examples and MCP calling conventions |
| [Workflow Patterns](references/workflow-patterns.md) | Domain resolution, company research, search + qualify pipelines |
| [Domain Resolution](references/domain-resolution.md) | Decision tree for resolving ambiguous company domains |

## Data & Privacy

- **Data sources:** Aggregated from publicly available sources (business directories, social profiles, corporate websites)
- **Query logging:** Logged for service improvement and abuse prevention. Not shared with third parties
- **Your responsibility:** Use retrieved contact data in compliance with local laws (GDPR, CAN-SPAM, etc.)
- [Privacy Policy](https://lessie.ai/privacy) · [Terms of Service](https://lessie.ai/terms-of-service)

## Troubleshooting

**CLI not found?** `npm install -g @lessie/cli`

**Auth expired?** `lessie auth` — reopens the browser for login

**Skill not showing up?** Make sure your CLAUDE.md references the lessie skill. Add:

```
## lessie
Use /lessie for all people search, contact enrichment, company research, and sourcing tasks.
```

**Codex can't find the skill?** Check that `SKILL.md` exists at `.agents/skills/lessie/SKILL.md` or `~/.codex/skills/lessie/SKILL.md`

## Links

- Website: [lessie.ai](https://lessie.ai)
- Pricing: [lessie.ai/pricing](https://lessie.ai/pricing)
- Privacy: [lessie.ai/privacy](https://lessie.ai/privacy)
- Terms: [lessie.ai/terms-of-service](https://lessie.ai/terms-of-service)
