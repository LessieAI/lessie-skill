# Workflow Patterns

## 1. Search people at a company (company name only, domain unknown)

**Always resolve the company domain first.** Domain accuracy directly determines search quality.

Priority chain: `web_search` result → `enrich_organization` verification → agent knowledge (lowest)

**CLI mode:**
```bash
# Step 1: Find domain
lessie web-search --query 'CompanyName official website' --count 3

# Step 2: Verify domain
lessie enrich-org --domains '["candidate.com"]'

# Step 3: Search people with verified domain
lessie find-people \
  --filter '{"person_titles":["CTO"]}' \
  --checkpoint 'CTOs at CompanyName' \
  --domain '["verified_domain.com"]'
```

**MCP mode:**
1. `web_search("CompanyName official website")` → extract candidate domain
2. `enrich_organization(domains=["candidate.com"])` → verify
3. `find_people(filter={...}, website_domain=["verified_domain"])`

See [domain-resolution.md](domain-resolution.md) for the full decision tree and examples.

**Validation:**
- After Step 2, verify the enriched company name matches the user's intent. If the returned company is different, stop and confirm with the user before proceeding.
- After Step 3, if 0 results → try relaxing filters (remove location or broaden titles). If still 0, the domain may be wrong — go back to Step 1 with alternative search terms.

## 2. Search people at a company (domain already known)

```bash
lessie enrich-org --domains '["stripe.com"]'
lessie find-people --filter '{"person_titles":["Engineer"]}' --checkpoint 'Engineers at Stripe' --domain '["stripe.com"]'
lessie enrich-people --people '[{"first_name":"Jane","last_name":"Doe","domain":"stripe.com"}]'
```

**Validation:**
- If `enrich-org` returns a different primary domain than expected, use the enriched domain for `find-people`.
- If `find-people` returns fewer results than expected, try `--strategy saas_only` or broaden the title filter.

## 3. Research a company

```bash
lessie enrich-org --domains '["openai.com"]'
lessie job-postings --org-id '...'          # from enrich result
lessie company-news --org-ids '["..."]'
lessie find-people --filter '{"person_seniorities":["c_suite"]}' --checkpoint 'OpenAI leadership' --domain '["openai.com"]'
```

**Validation:**
- If `enrich-org` returns no result, fall back to `web_search` + `web_fetch` to gather company info manually.
- If `job-postings` or `company-news` returns empty, inform the user — the company may not have active postings or recent coverage in Lessie's data sources.

## 4. Verify a specific person

```bash
lessie enrich-people --people '[{"first_name":"Sam","last_name":"Altman","domain":"openai.com"}]'
# If no match:
lessie web-search --query 'Sam Altman OpenAI LinkedIn'
lessie web-fetch --url '...' --instruction 'Extract job title, company, and social links'
```

**Validation:**
- Cross-check: if `enrich-people` returns a match, verify the job title and company against the user's expectation. Enrichment data can be stale — if the user says "current CTO" but enrichment shows a different title, flag the discrepancy.
- If both enrichment and web search fail, inform the user the person could not be verified rather than guessing.

## 5. Search + qualify (find → self-judge or review)

1. `lessie find-people --filter '...' --checkpoint '...'` → returns `search_id` and `people`

2. **Triage the results yourself first.** Scan the returned profiles against the user's criteria:
   - **Obviously good** (title/company/location clearly match) → keep directly, no deep review needed
   - **Obviously bad** (wrong industry, wrong country, clearly irrelevant) → discard directly
   - **Ambiguous** (partially matching, unclear fit, need deeper verification) → send to `review_people`

3. Only call review for the ambiguous subset:
   ```bash
   lessie review-people --search-id '...' --person-ids '[...]' --checkpoints '[...]'
   ```
   This runs deep web research per person — takes 1-3 min, so skip it when you can already tell.

**Validation:**
- After triage, present results to the user grouped by confidence level (strong match / needs review / excluded) before running `review_people`, so the user can adjust criteria if needed.
- If `find-people` returns 0 results, do NOT immediately call review. Instead: relax filters (broader titles, remove location), try `--strategy saas_only`, or verify the domain is correct.
