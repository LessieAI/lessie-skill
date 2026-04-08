# CLI Tool Calling Reference

Output is always JSON on stdout, status messages on stderr. Use `--pretty` for human-readable formatting.

---

## lessie find-people

Search people from Lessie's 275M+ database by title, company, location, seniority, or social audience.

### Commands

```bash
# B2B search by title and location
lessie find-people \
  --filter '{"person_titles":["CTO"],"person_locations":["United States"]}' \
  --checkpoint 'CTOs at US tech companies'

# Scoped to a known domain
lessie find-people \
  --filter '{"person_titles":["Engineering Manager"]}' \
  --checkpoint 'EMs at Stripe' \
  --domain '["stripe.com"]'

# With seniority and exclusions
lessie find-people \
  --filter '{"person_titles":["ML Engineer"],"person_seniorities":["senior"],"person_not_titles":["Intern"]}' \
  --checkpoint 'Senior ML engineers' \
  --target-count 20

# KOL / influencer search
lessie find-people \
  --filter '{"platform":"instagram","follower_min":100000,"content_topics":["beauty"]}' \
  --checkpoint '美国美妆博主'

# Single person lookup by name + company
lessie find-people \
  --filter '{"person_name":"Sam Altman","company":"OpenAI"}' \
  --checkpoint 'Sam Altman at OpenAI'

# Exclude companies + add extra requirements
lessie find-people \
  --filter '{"person_titles":["CTO"],"person_locations":["US"]}' \
  --checkpoint 'CTOs with GitHub' \
  --exclude '{"companies":["Google"]}' \
  --extra 'Must have GitHub account'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--filter <json>` | Yes | JSON filter object (see filter fields below). Do NOT use `--title`/`--company` flags |
| `--checkpoint <text>` | Yes | Natural language search intent |
| `--strategy <s>` | No | `hybrid` (default, 60-120s), `saas_only` (30-60s), `saas_first` (40-90s), `web_only` (60-120s) |
| `--target-count <n>` | No | Target results count, 1-1000 (default: 50) |
| `--domain <json>` | No | Company domains for precision, e.g. `'["stripe.com"]'` |
| `--exclude <json>` | No | Exclusion conditions, e.g. `'{"companies":["Google"]}'` |
| `--extra <text>` | No | Extra natural language requirements |
| `--search-id <id>` | No | Custom search session ID (auto-generated if omitted) |
| `--language <lang>` | No | Results language, e.g. `en-US`, `zh-CN` (default: `en-US`) |

### Filter fields (B2B)

| Field | Type | Example |
|-------|------|---------|
| `person_titles` | `string[]` | `["CTO", "VP Engineering"]` |
| `person_not_titles` | `string[]` | `["Intern", "Assistant"]` |
| `person_locations` | `string[]` | `["United States", "London"]` |
| `person_seniorities` | `string[]` | `["c_suite", "vp", "director"]` |
| `contact_email_status` | `string[]` | `["verified", "guessed", "unavailable"]` |
| `organization_locations` | `string[]` | `["San Francisco", "China"]` |
| `organization_num_employees_ranges` | `string[]` | `["51,200"]` |

### Filter fields (single person)

| Field | Type | Example |
|-------|------|---------|
| `person_name` | `string` | `"Sam Altman"` |
| `company` | `string` | `"OpenAI"` |

### Filter fields (KOL / influencer)

| Field | Type | Example |
|-------|------|---------|
| `platform` | `string` | `instagram`, `youtube`, `tiktok`, `twitter`, ... |
| `follower_min` | `number` | `100000` |
| `content_topics` | `string[]` | `["beauty", "skincare"]` |

### Seniority values

`owner`, `founder`, `c_suite`, `partner`, `vp`, `head`, `director`, `manager`, `senior`, `entry`, `intern`

### Strategy reference

| Strategy | Speed | Best for |
|----------|-------|----------|
| `hybrid` | 60-120s | Default, highest recall |
| `saas_only` | 30-60s | Structured B2B queries, fastest |
| `saas_first` | 40-90s | SaaS first, fallback to web |
| `web_only` | 60-120s | Public figures, KOLs |

### AI usage guidance

- **Always resolve the company domain first** when the user provides only a company name. See [domain-resolution.md](domain-resolution.md).
- If `hybrid` times out or fails, retry with `--strategy saas_only`.
- If 0 results, try relaxing filters (remove location or broaden titles) before concluding the search failed.

---

## lessie enrich-people

Enrich known people with professional profiles (B2B) or KOL/influencer data.

Two mutually exclusive paths based on input:
- **KOL path**: If any `screen_name`/`username` is provided → enrich via social platform data
- **B2B path**: Otherwise → enrich via professional databases (cache-first)

### Commands

```bash
# B2B — by LinkedIn URL (best match rate, cache-first)
lessie enrich-people \
  --people '[{"linkedin_url":"https://linkedin.com/in/samaltman"}]'

# B2B — by name + domain
lessie enrich-people \
  --people '[{"first_name":"Sam","last_name":"Altman","domain":"openai.com"}]'

# B2B — by full name + organization
lessie enrich-people \
  --people '[{"name":"Sam Altman","organization_name":"OpenAI"}]'

# B2B — by email
lessie enrich-people \
  --people '[{"email":"sam@openai.com"}]'

# B2B — include personal emails
lessie enrich-people \
  --people '[{"first_name":"Sam","last_name":"Altman","domain":"openai.com"}]' \
  --personal-emails

# KOL — Twitter/X
lessie enrich-people --people '[{"twitter_screen_name":"elonmusk"}]'

# KOL — Instagram
lessie enrich-people --people '[{"instagram_username":"natgeo"}]'

# KOL — TikTok
lessie enrich-people --people '[{"tiktok_username":"charlidamelio"}]'

# KOL — YouTube
lessie enrich-people --people '[{"youtube_username":"MrBeast"}]'

# Batch (max 10 per call)
lessie enrich-people \
  --people '[{"linkedin_url":"https://linkedin.com/in/person1"},{"linkedin_url":"https://linkedin.com/in/person2"}]'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--people <json>` | Yes | JSON array of person objects (max 10 per call) |
| `--personal-emails` | No | Include personal emails (B2B path only, default: false) |

### Person object fields

| Path | Fields | Notes |
|------|--------|-------|
| B2B (best) | `linkedin_url` | Highest match rate, cache-first |
| B2B | `first_name`, `last_name`, `domain` | Good match rate |
| B2B | `name`, `organization_name` | Full name + company |
| B2B | `email` | Direct lookup |
| KOL — Twitter | `twitter_screen_name` | |
| KOL — Instagram | `instagram_username` | |
| KOL — TikTok | `tiktok_username` | |
| KOL — YouTube | `youtube_username` | |

### AI usage guidance

- Providing `domain` alongside name greatly improves B2B match accuracy.
- For batches larger than 10, split into multiple calls.
- Credits are only charged for successful matches.
- **Note**: the flag is `--personal-emails`, NOT `--include-personal-emails`.

---

## lessie review-people

Deep-review people from a previous `find-people` search via web research. Takes 1-3 minutes per person.

### Commands

```bash
# Basic review with one checkpoint
lessie review-people \
  --search-id 'mcp_abc123' \
  --person-ids '["6578a1b2c3d4e5f6","6578a1b2c3d4e5f7"]' \
  --checkpoints '[{"key":"relevance","title":"Relevance","description":"Is this person a senior ML engineer at a FAANG company?","category":"career"}]' \
  --query 'Find senior ML engineers at FAANG'

# Multiple checkpoints
lessie review-people \
  --search-id 'mcp_abc123' \
  --person-ids '["id1"]' \
  --checkpoints '[
    {"key":"title_match","title":"Title Match","description":"Does the person hold a CTO or VP Eng title?","category":"career"},
    {"key":"company_fit","title":"Company Fit","description":"Is the company an AI startup with <500 employees?","category":"company"}
  ]'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--search-id <id>` | Yes | Search ID from `find-people` result, e.g. `'mcp_abc123'` |
| `--person-ids <json>` | Yes | JSON array of person IDs to review |
| `--checkpoints <json>` | Yes | JSON array of checkpoint objects (see format below) |
| `--query <text>` | No | Original user query for context |

### Checkpoint object

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `key` | Yes | `string` | Identifier string, e.g. `"relevance"` |
| `title` | Yes | `string` | Display title, e.g. `"Relevance"` |
| `description` | Yes | `string` | English description for the reviewer LLM |
| `category` | Yes | `string` | One of: `company`, `school`, `career`, `other` |

### AI usage guidance

- Only use for **ambiguous** candidates. Skip for obvious matches or mismatches — triage results first.
- If `find-people` returned 0 results, do NOT call review. Instead relax filters or verify the domain.

---

## lessie find-orgs

Discover companies by name, keyword, location, size, funding, and hiring activity. At least one search option is required.

### Commands

```bash
# By name
lessie find-orgs --name 'OpenAI'

# By keyword and location
lessie find-orgs --keyword-tags '["AI","SaaS"]' --locations '["United States"]'

# By size and funding
lessie find-orgs --employees '["51,200"]' --funding '{"min":1000000}'

# By hiring activity
lessie find-orgs --job-titles '["Engineer"]' --num-jobs '{"min":5}'

# With revenue filter and pagination
lessie find-orgs \
  --keyword-tags '["fintech"]' \
  --revenue '{"min":300000,"max":50000000}' \
  --page 2 --per-page 50

# Exclude locations
lessie find-orgs \
  --keyword-tags '["AI"]' \
  --locations '["Asia"]' \
  --not-locations '["Russia"]'

# By latest funding date
lessie find-orgs \
  --keyword-tags '["AI"]' \
  --latest-funding '{"min":500000}' \
  --funding-date '{"min":"2025-01-01"}'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name <text>` | No* | Company name (fuzzy match) |
| `--keyword-tags <json>` | No* | Keyword tags, e.g. `'["AI","SaaS"]'` |
| `--domains <json>` | No* | Filter by known domains |
| `--locations <json>` | No | HQ locations |
| `--not-locations <json>` | No | Exclude HQ locations |
| `--employees <json>` | No | Employee count ranges (see below) |
| `--revenue <json>` | No | Revenue range (USD), e.g. `'{"min":300000,"max":50000000}'` |
| `--funding <json>` | No | Total funding range (USD) |
| `--latest-funding <json>` | No | Latest round amount range (USD) |
| `--funding-date <json>` | No | Latest funding date range (ISO format) |
| `--job-titles <json>` | No | Active job title keywords |
| `--job-locations <json>` | No | Active job locations |
| `--num-jobs <json>` | No | Active jobs count range |
| `--job-posted-at <json>` | No | Job posting date range (ISO format) |
| `--page <n>` | No | Page number (default: 1) |
| `--per-page <n>` | No | Results per page (default: 25, max: 100) |

*At least one of `--name`, `--keyword-tags`, or `--domains` is required.

### Employee count ranges

`"1,10"` (Micro), `"11,50"` (Small), `"51,200"` (Medium), `"201,500"`, `"501,1000"`, `"1001,5000"` (Large), `"5001,10000"` (Enterprise)

### Range format

- Numeric: `{"min": 1000000}` or `{"min": 1000000, "max": 5000000}`
- Date: `{"min": "2025-01-01"}` or `{"min": "2025-01-01", "max": "2025-12-31"}`

---

## lessie enrich-org

Enrich known organizations by domain. Returns full profile including industry, employees, funding, tech stack, and `organization_id` needed by `job-postings` and `company-news`.

### Commands

```bash
# Single domain (string shorthand)
lessie enrich-org --domains stripe.com

# Multiple domains (JSON array, max 10)
lessie enrich-org --domains '["stripe.com","openai.com"]'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--domains <string\|json>` | Yes | Company domain(s). Accepts a single string or JSON array (max 10) |

---

## lessie job-postings

Get active job openings for a company. Requires `organization_id` from `enrich-org` or `find-orgs`.

### Commands

```bash
lessie job-postings --org-id '5f5e100abc123'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--org-id <id>` | Yes | Organization ID (from `enrich-org` or `find-orgs` result) |

---

## lessie company-news

Search news articles for companies. Requires `organization_id` from `enrich-org` or `find-orgs`.

### Commands

```bash
# Single company
lessie company-news --org-ids '["5f5e100abc123"]'

# Multiple companies with pagination
lessie company-news --org-ids '["id1","id2"]' --page 2 --per-page 10
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--org-ids <json>` | Yes | JSON array of organization IDs |
| `--page <n>` | No | Page number (default: 1) |
| `--per-page <n>` | No | Results per page (default: 25) |

---

## lessie web-search

Search the open web. Cached results make follow-up `web-fetch` calls on the same URLs free.

### Commands

```bash
lessie web-search --query 'AI startups 2026'
lessie web-search --query 'OpenAI funding rounds' --count 5
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--query <text>` | Yes | Search query string |
| `--count <n>` | No | Max results (default: 10) |

### AI usage guidance

- If a result has `has_content: true`, calling `web-fetch` on that URL is instant (cached, no extra cost).

---

## lessie web-fetch

Fetch a web page and extract specific information via AI summarization.

### Commands

```bash
lessie web-fetch --url 'https://openai.com/about' --instruction 'Extract the leadership team'
lessie web-fetch --url 'https://stripe.com/jobs' --instruction 'List all engineering positions'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--url <url>` | Yes | URL to fetch |
| `--instruction <text>` | Yes | What to extract from the page |

---

## Generic tool call (fallback)

`lessie tools` and `lessie call` are the universal fallback for any tool — including when shortcuts fail due to argument changes or server-side updates.

### Commands

```bash
# List all available remote tools with full parameter schemas
lessie tools
lessie tools --pretty
lessie tools | jq '.[].name'

# Call any tool by name with raw JSON arguments
lessie call find_people --args '{"filter":{"person_titles":["CTO"]},"checkpoint_context":"CTOs"}'
lessie call web_search --args '{"query":"AI startups"}'
```

### When to use

- A shortcut command doesn't exist for a tool
- The remote server has added new tools not yet covered by CLI shortcuts
- You need to pass parameters that the shortcut doesn't expose

### Error recovery: fall back to `tools` + `call`

**When a CLI shortcut command fails 3+ times** (argument errors, schema mismatches, unexpected parameters), stop retrying the shortcut and switch to this workflow:

1. Run `lessie tools` to fetch the **current** remote tool schema.
2. Find the matching tool name (e.g., `find_people` for `lessie find-people`).
3. Read its `inputSchema` to understand the exact parameter names, types, and required fields.
4. Call it via `lessie call <tool_name> --args '{ ... }'` using the correct schema.

```bash
# Example: shortcut fails repeatedly
lessie web-fetch --url 'https://...' --instruction '...'
# Error: --url is required. URL to fetch   (3rd failure)

# Fall back to tools + call
lessie tools                                    # Check current schema for web_fetch
lessie call web_fetch --args '{"url":"https://...","instruction":"..."}'
```
