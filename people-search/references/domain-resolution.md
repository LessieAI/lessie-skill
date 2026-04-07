# Domain Resolution

When a user mentions a company name without providing its domain, you must resolve the domain before calling `find_people`. An incorrect domain means zero or wrong results.

## Priority chain

1. **`enrich_organization` verified domain** — highest confidence
2. **`web_search` multi-source confirmed domain** — reliable when enrichment has no match
3. **Agent's own knowledge** — lowest priority, only as last resort

## Decision tree

```
User mentions company name
│
├─ Domain already known (e.g. user said "stripe.com")
│   └─ Go directly to enrich_organization → find_people
│
└─ Domain unknown
    │
    ├─ Step 1: web_search("[company] official website")
    │   └─ Extract candidate domain from results
    │
    ├─ Step 2: enrich_organization(domains=["candidate_domain"])
    │   ├─ Found → use the returned domain (may differ from candidate)
    │   └─ Not found → Step 3
    │
    ├─ Step 3: Check web_search results
    │   ├─ Multiple sources agree (official site, LinkedIn, etc.) → use that domain
    │   └─ Ambiguous → ask user to confirm
    │
    └─ Step 4: find_people(website_domain=["resolved_domain"], ...)
```

## Examples

**Scenario A: enrichment returns a different domain**

1. `web_search("Havelsan official website")` → havelsan.com.tr
2. `enrich_organization(domains=["havelsan.com.tr"])` → returns domain: "havelsan.com"
3. Use `havelsan.com` (enrichment result takes precedence)
4. `find_people(website_domain=["havelsan.com"], filter={...})`

**Scenario B: enrichment has no match**

1. `web_search("Havelsan official website")` → havelsan.com.tr
2. `enrich_organization(domains=["havelsan.com.tr"])` → not found
3. Web results consistently point to havelsan.com.tr → adopt it
4. `find_people(website_domain=["havelsan.com.tr"], filter={...})`

## When to skip

For widely recognized domains (google.com, microsoft.com, apple.com), you may skip the full resolution flow and go directly to `enrich_organization` for verification.
