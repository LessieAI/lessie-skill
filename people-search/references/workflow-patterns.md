# Workflow Patterns

## 1. Search people at a company (company name ambiguous)

If the company name could be multiple entities (`Manus` → Manus AI / Manus Bio / Manus Plus / ...), disambiguate first. Otherwise the agent searches the wrong company and you waste 20 credits.

**CLI:**
```bash
# Step 1: find the candidate domain
lessie web-search --query 'CompanyName official website' --count 3

# Step 2: verify it's the right company
lessie enrich-org --domains '["candidate.com"]'

# Step 3: now pass the resolved company + domain into the NL query
lessie find-people --query "CTOs at CompanyName (candidate.com)" --target-count 10
```

See [domain-resolution.md](domain-resolution.md) for the full decision tree.

**Validation:**
- After Step 2, verify the enriched company name matches the user's intent. If the returned company is different, stop and confirm with the user before proceeding.
- After Step 3, 0 results → the company name + domain in the query may still be ambiguous on the agent side. Try a more specific query (add location, industry context).

## 2. Search people at a company (domain already known)

```bash
lessie enrich-org --domains '["stripe.com"]'                        # optional verify
lessie find-people --query "10 engineers at Stripe (stripe.com)" --target-count 10
lessie enrich-people --people '[{"first_name":"Jane","last_name":"Doe","domain":"stripe.com"}]'
```

Just put the company name + domain into the natural-language query. The agent uses them as filter inputs internally.

**Validation:**
- If `enrich-org` returns a different primary domain than expected, use the enriched domain in the `find-people` query.
- If `find-people` returns fewer results than expected, run a second call with a broader query (drop location or seniority qualifiers). Don't retry the exact same query — it'll return the same results.

## 3. Research a company

```bash
lessie enrich-org --domains '["openai.com"]'
lessie job-postings --org-id '...'          # from enrich result
lessie company-news --org-ids '["..."]'
lessie find-people --query "OpenAI c-suite leadership (openai.com)" --target-count 10
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

1. `lessie find-people --query '<user's task verbatim>'` → returns `search_id` and `people`

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
- If `find-people` returns 0 results, do NOT immediately call review. Instead: re-run with a broader NL query (drop location / industry qualifiers, or describe the role more loosely), or verify the company name needs disambiguation first.

## 6. Unlock contact emails (from your own search results)

After `find-people`, the emails in the response are **masked** (`****@domain.com`). To get the actual email, call `unlock_emails` with the `search_id` + the `person_id` of each person you want.

```bash
# Step 1: find — capture search_id + person_ids
lessie find-people --query "10 CTOs at tech companies" --target-count 10
# response → { "search_id": "mcp_...", "people": [{"person_id":"...", ...}, ...] }

# Step 2: unlock — pass the search_id + a subset of person_ids
lessie unlock-emails \
  --search-id <from-step-1> \
  --person-ids '["<id-of-person-1>","<id-of-person-2>"]'
```

**Use `unlock_emails`, not `enrich-people`, for this flow.** `enrich-people` re-queries the underlying providers and bills again; `unlock_emails` reuses your unlock history (same person, any prior search → free).

**Validation:**
- The `summary` field reports `newly_unlocked` (charged) vs `already_unlocked` (free). Cross-check `points_deducted = newly_unlocked × price_per_unlock`.
- `non_unlockable` persons have no enrichable handle on file — there is no way to get their email; do NOT retry.
- Hard cap of 50 person_ids per call — split larger lists.

## 7. Unlock email by handle (no prior search)

When the user gives you a LinkedIn URL / Twitter username / Instagram handle directly — and you have **not** run `find-people` for that person — use `unlock_email_by_handle`.

```bash
# Single handle — user pasted a LinkedIn URL, you extracted the handle
lessie unlock-email-by-handle \
  --handles '[{"platform":"linkedin","handle":"samaltman"}]'

# Batch (max 10) across platforms
lessie unlock-email-by-handle \
  --handles '[{"platform":"linkedin","handle":"samaltman"},{"platform":"twitter","handle":"elonmusk"}]'
```

**Critical: NOT idempotent.** Server does no dedup. Calling the same `(platform, handle)` twice charges twice. **Track within your conversation which handles you've already resolved** and skip re-queries.

**Pass bare handles, NOT URLs.** Parse the URL yourself:
- `https://linkedin.com/in/samaltman` → `{"platform":"linkedin","handle":"samaltman"}`
- `https://twitter.com/elonmusk` → `{"platform":"twitter","handle":"elonmusk"}`
- `https://instagram.com/natgeo/` → `{"platform":"instagram","handle":"natgeo"}`

**Validation:**
- Verify the handle once before bulk: a typo costs nothing if it resolves to `not_found`, but if it accidentally resolves to a real different person's email you pay for the wrong result.
- If the user has any historical `search_id` containing this person, prefer Pattern 6 (`unlock_emails`) — it dedups across all your searches.
