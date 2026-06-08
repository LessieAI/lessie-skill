# CLI Tool Calling Reference

Output is always JSON on stdout, status messages on stderr. Use `--pretty` for human-readable formatting.

---

## lessie find-people

Find people matching a **natural-language task**. Pass the user's request verbatim through `--query`. The remote agent picks data sources (B2B / KOL / web), keywords, and decides when to stop. Hard ceiling of **3 tool calls + 60s wall-clock budget** per request bounds worst-case latency.

### Commands

```bash
# B2B
lessie find-people --query "10 senior engineers at Google in the Bay Area"

# Scoped to a known company / domain — just include it in the query
lessie find-people --query "Engineering Managers at Stripe (stripe.com)"

# With seniority + exclusions — phrase naturally
lessie find-people \
  --query "Senior ML engineers at AI companies, excluding interns and assistants" \
  --target-count 20

# KOL / influencer
lessie find-people --query "Instagram beauty creators with 100k+ followers in the US"

# Single person lookup
lessie find-people --query "Sam Altman at OpenAI"

# Niche / industry-specific
lessie find-people --query "CTOs of AI startups raising Series B"

# Non-English query is fine; pass the user's words verbatim and set --language to match
lessie find-people --query "Engenheiros de machine learning em São Paulo" --language pt-BR
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--query <text>` | Yes | Natural-language task, 1-2000 chars. Pass the user's request verbatim — niche industry terms are search-quality-critical |
| `--target-count <n>` | No | How many results you want, 1-100 (default: 30). Used as a stop signal — agent stops when results >= this count |
| `--language <lang>` | No | Output language, e.g. `en-US`, `zh-CN` (default: `en-US`) |

### Response shape

```json
{
  "success": true,
  "search_id": "mcp_abc123",
  "people": [...],
  "total_found": N,
  "after_hard_filter": N,
  "elapsed_seconds": 12.3,
  "strategy_used": "hybrid",
  "checkpoints_used": [],
  "sources_used": [],
  "points_deducted": 20,
  "partial": false  // only present + true if the 60s budget fired
}
```

When `"partial": true`, the response also carries `"timeout_reason": "wall_clock_60s"` — the agent hit the 60s cap; results are whatever was collected before timeout.

### AI usage guidance

- **Pass the user's words verbatim.** Don't paraphrase niche industry terms — "fintech" stays "fintech", not "financial tech". The remote agent's search quality depends on those exact words being in the query.
- **Pre-resolve ambiguous company names** before searching. If the user mentions a company that could be multiple entities (`Manus` → Manus AI / Manus Bio / Manus Plus / ...), run `lessie enrich-org --domains '[...]'` first or ask the user to disambiguate. Include the resolved domain in the query (e.g. `"... at Manus AI (manus.im)"`).
- **0 results** → check whether the company name needs disambiguation, or restate the task with broader phrasing. Don't keep retrying the same query.
- **`partial: true`** → the agent hit the 60s budget. Results are usable but incomplete; consider re-running with a tighter `target_count` or a more specific query.

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

## lessie unlock-emails

Unlock email addresses for people from a previous `lessie find-people` result. **Per-user idempotent**: people you've already unlocked (in this search or any other) are returned at 0 cost. Only newly unlocked emails are charged.

### Commands

```bash
# Basic — unlock 3 persons from a recent search
lessie unlock-emails \
  --search-id mcp_abc123 \
  --person-ids '["6578a1b2c3d4e5f6","6578a1b2c3d4e5f7","6578a1b2c3d4e5f8"]'

# After find-people, capture search_id + person_ids then unlock
lessie find-people --query "3 CTOs" --target-count 3
# → take the search_id from the response and the person_id of each person
lessie unlock-emails --search-id <search_id> --person-ids '["<id1>","<id2>","<id3>"]'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--search-id <id>` | Yes | Search ID from a prior `find-people` response you own |
| `--person-ids '[...]'` | Yes | JSON array of `person_id` values (the `person_id` field of each `find-people` result), **1–50 items** |

### Response

```json
{
  "success": true,
  "search_id": "mcp_abc123",
  "results": [
    {"person_id": "6578…6", "status": "newly_unlocked", "email": "alice@stripe.com"},
    {"person_id": "6578…7", "status": "already_unlocked", "email": "bob@stripe.com"},
    {"person_id": "6578…8", "status": "non_unlockable", "email": null, "reason": "no_enrichable_handle"}
  ],
  "summary": {"total_requested": 3, "already_unlocked": 1, "newly_unlocked": 1, "non_unlockable": 1, "failed": 0},
  "points_deducted": 1,
  "price_per_unlock": 1
}
```

Per-person `status` values:
- `newly_unlocked` — email resolved for the first time; charged
- `already_unlocked` — you've unlocked this person before; free
- `non_unlockable` — no enrichable handle on this row; free
- `failed` — resolution attempted but failed (API error, etc.); free
- `not_in_search` — the `person_id` doesn't belong to this `search_id`; free

### AI usage guidance

- **Use this — not `enrich-people` — to get emails from your own search results.** `enrich-people` will re-query the underlying providers and bill again; `unlock_emails` reuses your unlock history.
- After `find-people`, you'll have `search_id` + each person's `person_id` directly in the response. Pass them through.
- If you hit the 50-person cap, split into batches.
- Cross-search idempotency: if you unlocked the same person in a prior search and they appear again in a new search, the second unlock is still free.

---

## lessie unlock-email-by-handle

Unlock email addresses for known social handles, **without requiring a prior `find-people` search**. Pass a list of `{platform, handle}` pairs. Charged per successful unlock; `not_found` and `failed` are free.

**Important**: this tool is **NOT idempotent**. Repeated calls on the same `(platform, handle)` will re-charge each time it resolves. Use `unlock-emails` instead when the person came from a `find-people` you ran.

### Commands

```bash
# Single handle
lessie unlock-email-by-handle \
  --handles '[{"platform":"linkedin","handle":"samaltman"}]'

# Batch (max 10) — mix platforms
lessie unlock-email-by-handle \
  --handles '[{"platform":"linkedin","handle":"samaltman"},{"platform":"twitter","handle":"elonmusk"},{"platform":"instagram","handle":"natgeo"}]'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--handles '[...]'` | Yes | JSON array of HandleSpec, **1–10 items**. Each item: `{"platform": <one of linkedin/youtube/instagram/tiktok/twitter>, "handle": <bare handle string>}` |

### HandleSpec fields

| Field | Type | Notes |
|-------|------|-------|
| `platform` | `string` (enum) | One of `linkedin`, `youtube`, `instagram`, `tiktok`, `twitter` — **lowercase** |
| `handle` | `string` | The **bare handle**, NOT a URL. For LinkedIn pass the `/in/<handle>` segment (e.g. `samaltman`), for Twitter the screen name, for Instagram/TikTok/YouTube the username. 1–200 chars |

### Response

```json
{
  "success": true,
  "results": [
    {"platform": "linkedin", "handle": "samaltman", "status": "unlocked", "email": "sam@openai.com"},
    {"platform": "instagram", "handle": "fake_xyz", "status": "not_found", "email": null},
    {"platform": "tiktok", "handle": "weird", "status": "failed", "email": null, "reason": "enrich_exception"}
  ],
  "summary": {"total_requested": 3, "unlocked": 1, "not_found": 1, "failed": 1},
  "points_deducted": 1,
  "price_per_unlock": 1
}
```

Per-handle `status`:
- `unlocked` — email resolved; charged
- `not_found` — no email available for this handle; free
- `failed` — exception during resolution; free

### AI usage guidance

- **Pass bare handles, never URLs.** `samaltman`, not `https://linkedin.com/in/samaltman`. The agent should parse the URL itself before calling.
- **Verify the handle once before bulk operations.** Because of non-idempotency, a wrong handle costs you nothing on `not_found` — but if it accidentally resolves to a real different person's email, you pay for the wrong result. Spot-check first.
- **Prefer `unlock_emails` when applicable.** If you ran `find-people` and the person was in the results, that flow's idempotency saves credits. Use `unlock_email_by_handle` only when the handle wasn't surfaced by a search.
- **Cache results in your turn's context.** Since the server doesn't dedup, *you* should: track which `(platform, handle)` pairs you've already resolved this conversation and not re-query.

---

## lessie unlock-phones

Unlock company phone numbers for people from a previous `lessie find-people` result. **Per-user idempotent**: people you've already unlocked (in this search or any other) are returned at 0 cost. Only newly unlocked phones are charged.

The contract is symmetric to `lessie unlock-emails` — same `search_id` + `person_ids` shape, same idempotency rules, same `status` values. Use `unlock-emails` to get the email and `unlock-phones` to get the phone; the two are independent and charged separately.

### Commands

```bash
# Basic — unlock phones for 3 persons from a recent search
lessie unlock-phones \
  --search-id mcp_abc123 \
  --person-ids '["6578a1b2c3d4e5f6","6578a1b2c3d4e5f7","6578a1b2c3d4e5f8"]'

# After find-people, capture search_id + person_ids then unlock
lessie find-people --query "3 CTOs" --target-count 3
# → take the search_id from the response and the person_id of each person
lessie unlock-phones --search-id <search_id> --person-ids '["<id1>","<id2>","<id3>"]'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--search-id <id>` | Yes | Search ID from a prior `find-people` response you own |
| `--person-ids '[...]'` | Yes | JSON array of `person_id` values, **1–50 items** |

### Response

```json
{
  "success": true,
  "search_id": "mcp_abc123",
  "results": [
    {"person_id": "6578…6", "status": "newly_unlocked", "phone": "+1-415-555-0101"},
    {"person_id": "6578…7", "status": "already_unlocked", "phone": "+1-650-555-0102"},
    {"person_id": "6578…8", "status": "non_unlockable", "phone": null, "reason": "no_enrichable_handle"}
  ],
  "summary": {"total_requested": 3, "already_unlocked": 1, "newly_unlocked": 1, "non_unlockable": 1, "failed": 0},
  "points_deducted": 3,
  "price_per_unlock": 3
}
```

Per-person `status` values (same as `unlock-emails`):
- `newly_unlocked` — phone resolved for the first time; charged
- `already_unlocked` — you've unlocked this person before; free
- `non_unlockable` — no enrichable handle on this row; free
- `failed` — resolution attempted but failed; free
- `not_in_search` — the `person_id` doesn't belong to this `search_id`; free

### AI usage guidance

- **Don't unlock the phone if the user only wants email.** They are two separate charges. Pick the channel the user actually needs.
- **Phone enrichment is currently best for LinkedIn-based search results.** B2B searches that resolved a LinkedIn handle for the person tend to have higher phone hit rates; KOL / social-only results often come back `non_unlockable`.
- After `find-people`, you'll have `search_id` + each person's `person_id` directly in the response. Pass them through.
- If you hit the 50-person cap, split into batches.
- Cross-search idempotency: same person, any prior search → second unlock is free.

---

## lessie unlock-phone-by-handle

Unlock phone numbers for known social handles, **without requiring a prior `find-people` search**. Pass a list of `{platform, handle}` pairs.

**Important platform constraint:** **Only `platform="linkedin"` actually resolves** (via ContactOut). Handles for other platforms (`twitter` / `instagram` / `tiktok` / `youtube`) return `status="not_found"` with `reason="unsupported_platform"` and are **NOT charged**. This is by design — the server short-circuits non-LinkedIn handles before making any API call.

**Also important**: this tool is **NOT idempotent**. Repeated calls on the same `(linkedin, handle)` will re-charge each time it resolves. Use `unlock-phones` instead when the person came from a `find-people` you ran.

### Commands

```bash
# Single LinkedIn handle
lessie unlock-phone-by-handle \
  --handles '[{"platform":"linkedin","handle":"samaltman"}]'

# Batch (max 10) — all LinkedIn (other platforms will all return unsupported_platform)
lessie unlock-phone-by-handle \
  --handles '[{"platform":"linkedin","handle":"samaltman"},{"platform":"linkedin","handle":"ilya-sutskever"}]'
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--handles '[...]'` | Yes | JSON array of HandleSpec, **1–10 items**. Each item: `{"platform": <one of linkedin/youtube/instagram/tiktok/twitter>, "handle": <bare handle string>}`. **Only `linkedin` resolves** — other platforms return `not_found` + `reason="unsupported_platform"`, free of charge |

### HandleSpec fields

| Field | Type | Notes |
|-------|------|-------|
| `platform` | `string` (enum) | Schema accepts `linkedin`, `youtube`, `instagram`, `tiktok`, `twitter` — but **only `linkedin` resolves to a phone today**. Others always return unsupported |
| `handle` | `string` | The **bare handle**, NOT a URL. For LinkedIn pass the `/in/<handle>` segment (e.g. `samaltman`). 1–200 chars |

### Response

```json
{
  "success": true,
  "results": [
    {"platform": "linkedin", "handle": "samaltman", "status": "unlocked", "phone": "+1-415-555-0101"},
    {"platform": "linkedin", "handle": "fake_xyz", "status": "not_found", "phone": null},
    {"platform": "twitter", "handle": "elonmusk", "status": "not_found", "phone": null, "reason": "unsupported_platform"}
  ],
  "summary": {"total_requested": 3, "unlocked": 1, "not_found": 2, "failed": 0},
  "points_deducted": 3,
  "price_per_unlock": 3
}
```

Per-handle `status`:
- `unlocked` — phone resolved; charged
- `not_found` — no phone available for this handle (or non-LinkedIn platform → check `reason`); free
- `failed` — exception during resolution; free

### AI usage guidance

- **Tell the user up-front when their handle isn't LinkedIn.** A Twitter handle will always come back `unsupported_platform`; don't appear to "try and fail" — say "phone lookup currently only works for LinkedIn handles; for Twitter you can try `unlock-email-by-handle` instead."
- **Pass bare handles, never URLs.** `samaltman`, not `https://linkedin.com/in/samaltman`. The agent should parse the URL itself before calling.
- **Verify the handle once before bulk operations.** A typo costs nothing on `not_found`, but if it accidentally resolves to a real different person's phone, you pay for the wrong result.
- **Prefer `unlock_phones` when applicable.** If you ran `find-people` and the person was in the results, that flow's idempotency saves credits. Use `unlock_phone_by_handle` only when the handle wasn't surfaced by a search.
- **Cache results in your turn's context.** The server doesn't dedup — track which LinkedIn handles you've already resolved.

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
lessie call find_people --args '{"query":"10 CTOs at AI startups"}'
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
