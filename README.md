# Lessie Skills

> AI-native people search and enrichment — as agent skills.

This repo contains skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Codex](https://github.com/openai/codex), and any agent that supports the [SKILL.md standard](https://github.com/anthropics/claude-code).

## Get Started

1. **Create a free account** at [app.lessie.ai](https://app.lessie.ai/login?utm_source=github&utm_channel=skill) — new accounts include trial credits
2. **Install all skills:**

```bash
npx skills add LessieAI/lessie-skill -y -g
```

3. **Restart your agent** and run any skill by name

## Available Skills

| Skill | What it does |
|-------|--------------|
| [/people-search](people-search/) | Find people, enrich contacts, research companies, web search |

## See it work

```
You:    /people-search

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

## Alternative: MCP Server (no CLI needed)

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

## Uninstall

- **CLI + Skills:** `npm uninstall -g @lessie/cli && rm -rf ~/.lessie/ ~/.claude/skills/people-search`
- **MCP:** Remove the `"lessie"` entry from your MCP config and `rm -rf ~/.lessie/`
- **Codex:** `rm -rf ~/.codex/skills/people-search` (or `.agents/skills/people-search`)

## Credits & Pricing

Lessie is credit-based. New accounts get free trial credits. View balance and buy more at [lessie.ai/pricing](https://lessie.ai/pricing).

## Data & Privacy

- **Data sources:** Aggregated from publicly available sources (business directories, social profiles, corporate websites)
- **Query logging:** Logged for service improvement and abuse prevention. Not shared with third parties
- **Your responsibility:** Use retrieved contact data in compliance with local laws (GDPR, CAN-SPAM, etc.)
- [Privacy Policy](https://lessie.ai/privacy) · [Terms of Service](https://lessie.ai/terms-of-service)

## Troubleshooting

**CLI not found?** `npm install -g @lessie/cli`

**Auth expired?** `lessie auth` — reopens the browser for login

**Skill not showing up?** Make sure your CLAUDE.md references the people-search skill.

## Links

- Website: [lessie.ai](https://lessie.ai)
- Pricing: [lessie.ai/pricing](https://lessie.ai/pricing)
- Privacy: [lessie.ai/privacy](https://lessie.ai/privacy)
- Terms: [lessie.ai/terms-of-service](https://lessie.ai/terms-of-service)
