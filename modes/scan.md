# Mode: scan — Portal Scanner (Offer Discovery)

Scan configured job portals, filter by title relevance, and add new offers to the pipeline for later evaluation.

> **Note (v1.5+):** The default scanner (`scan.mjs` / `npm run scan`) is **zero-token** and only queries the public Greenhouse, Ashby, and Lever APIs directly. The Playwright/WebSearch tiers described below are the **agent** flow (run by Claude/Codex), not what `scan.mjs` does. If a company has no Greenhouse/Ashby/Lever API, `scan.mjs` will skip it; for those cases, the agent must manually complete Tier 1 (Playwright) or Tier 3 (WebSearch).

## Recommended execution

Run as a subagent to avoid consuming the main context:

```
Agent(
    subagent_type="general-purpose",
    prompt="[content of this file + specific data]",
    run_in_background=True
)
```

## Configuration

Read `portals.yml`, which contains:
- `search_queries`: List of WebSearch queries with `site:` filters per portal (broad discovery)
- `tracked_companies`: Specific companies with `careers_url` for direct navigation
- `title_filter`: positive/negative/seniority_boost keywords for title filtering

## Discovery strategy (3 tiers)

### Tier 1 — Direct Playwright (PRIMARY)

**For each company in `tracked_companies`:** Navigate to its `careers_url` with Playwright (`browser_navigate` + `browser_snapshot`), read ALL visible job listings, and extract title + URL for each one. This is the most reliable method because:
- Sees the page in real time (no cached Google results)
- Works with SPAs (Ashby, Lever, Workday)
- Detects new offers instantly
- Does not depend on Google indexing

**Every company MUST have `careers_url` in portals.yml.** If it does not, find it once, save it, and use it in future scans.

### Tier 2 — ATS APIs / Feeds (COMPLEMENTARY)

For companies with a public API or structured feed, use the JSON/XML response as a fast complement to Tier 1. It is faster than Playwright and reduces visual scraping errors.

**Current support (variables in `{}`):**
- **Greenhouse**: `https://boards-api.greenhouse.io/v1/boards/{company}/jobs`
- **Ashby**: `https://jobs.ashbyhq.com/api/non-user-graphql?op=ApiJobBoardWithTeams`
- **BambooHR**: list `https://{company}.bamboohr.com/careers/list`; offer detail `https://{company}.bamboohr.com/careers/{id}/detail`
- **Lever**: `https://api.lever.co/v0/postings/{company}?mode=json`
- **Teamtailor**: `https://{company}.teamtailor.com/jobs.rss`
- **Workday**: `https://{company}.{shard}.myworkdayjobs.com/wday/cxs/{company}/{site}/jobs`

**Parsing convention per provider:**
- `greenhouse`: `jobs[]` → `title`, `absolute_url`
- `ashby`: GraphQL `ApiJobBoardWithTeams` with `organizationHostedJobsPageName={company}` → `jobBoard.jobPostings[]` (`title`, `id`; build the public URL if not returned in the payload)
- `bamboohr`: list `result[]` → `jobOpeningName`, `id`; build the detail URL `https://{company}.bamboohr.com/careers/{id}/detail`; to read the full JD, GET the detail and use `result.jobOpening` (`jobOpeningName`, `description`, `datePosted`, `minimumExperience`, `compensation`, `jobOpeningShareUrl`)
- `lever`: root array `[]` → `text`, `hostedUrl` (fallback: `applyUrl`)
- `teamtailor`: RSS items → `title`, `link`
- `workday`: `jobPostings[]`/`jobPostings` (depending on tenant) → `title`, `externalPath` or URL built from the host

### Tier 3 — WebSearch queries (BROAD DISCOVERY)

The `search_queries` with `site:` filters cover portals across the board (all Ashby, all Greenhouse, etc.). Useful for discovering NEW companies not yet in `tracked_companies`, but results may be stale.

**Execution priority:**
1. Tier 1: Playwright → all `tracked_companies` with `careers_url`
2. Tier 2: API → all `tracked_companies` with `api:`
3. Tier 3: WebSearch → all `search_queries` with `enabled: true`

The tiers are additive — run them all, then merge and deduplicate the results.

## Workflow

1. **Read configuration**: `portals.yml`
2. **Read history**: `data/scan-history.tsv` → URLs already seen
3. **Read dedup sources**: `data/applications.md` + `data/pipeline.md`

4. **Tier 1 — Playwright scan** (parallel in batches of 3-5):
   For each company in `tracked_companies` with `enabled: true` and a defined `careers_url`:
   a. `browser_navigate` to the `careers_url`
   b. `browser_snapshot` to read all job listings
   c. If the page has filters/departments, navigate the relevant sections
   d. For each job listing extract: `{title, url, company}`
   e. If the page paginates results, navigate additional pages
   f. Accumulate into the candidate list
   g. If `careers_url` fails (404, redirect), try `scan_query` as fallback and flag for URL update

5. **Tier 2 — ATS APIs / feeds** (parallel):
   For each company in `tracked_companies` with `api:` defined and `enabled: true`:
   a. WebFetch the API/feed URL
   b. If `api_provider` is defined, use its parser; if not, infer from the domain (`boards-api.greenhouse.io`, `jobs.ashbyhq.com`, `api.lever.co`, `*.bamboohr.com`, `*.teamtailor.com`, `*.myworkdayjobs.com`)
   c. For **Ashby**, send POST with:
      - `operationName: ApiJobBoardWithTeams`
      - `variables.organizationHostedJobsPageName: {company}`
      - GraphQL query for `jobBoardWithTeams` + `jobPostings { id title locationName employmentType compensationTierSummary }`
   d. For **BambooHR**, the list only carries basic metadata. For each relevant item, read `id`, GET `https://{company}.bamboohr.com/careers/{id}/detail`, and extract the full JD from `result.jobOpening`. Use `jobOpeningShareUrl` as the public URL if present; otherwise, use the detail URL.
   e. For **Workday**, send a JSON POST with at least `{"appliedFacets":{},"limit":20,"offset":0,"searchText":""}` and paginate by `offset` until results are exhausted
   f. For each job extract and normalize: `{title, url, company}`
   g. Accumulate into the candidate list (dedup against Tier 1)

6. **Tier 3 — WebSearch queries** (parallel if possible):
   For each query in `search_queries` with `enabled: true`:
   a. Run WebSearch with the defined `query`
   b. From each result extract: `{title, url, company}`
      - **title**: from the result title (before " @ " or " | ")
      - **url**: result URL
      - **company**: after the " @ " in the title, or extract from the domain/path
   c. Accumulate into the candidate list (dedup against Tier 1+2)

6. **Filter by title** using `title_filter` from `portals.yml`:
   - At least 1 keyword from `positive` must appear in the title (case-insensitive)
   - 0 keywords from `negative` must appear
   - `seniority_boost` keywords give priority but are not required

7. **Deduplicate** against 3 sources:
   - `scan-history.tsv` → exact URL already seen
   - `applications.md` → company + normalized role already evaluated
   - `pipeline.md` → exact URL already pending or processed

7.5. **Verify liveness of WebSearch results (Tier 3)** — BEFORE adding to the pipeline:

   WebSearch results can be out of date (Google caches results for weeks or months). To avoid evaluating expired offers, verify every new URL from Tier 3 with Playwright. Tiers 1 and 2 are inherently real time and do not require this verification.

   For each new Tier 3 URL (sequential — NEVER Playwright in parallel):
   a. `browser_navigate` to the URL
   b. `browser_snapshot` to read the content
   c. Classify:
      - **Active**: role title visible + role description + visible Apply/Submit/Solicitar control within the main content. Do not count generic header/navbar/footer text.
      - **Expired** (any of these signals):
        - Final URL contains `?error=true` (Greenhouse redirects this way when the offer is closed)
        - Page contains: "job no longer available" / "no longer open" / "position has been filled" / "this job has expired" / "page not found"
        - Only navbar and footer visible, no JD content (content < ~300 chars)
   d. If expired: log in `scan-history.tsv` with status `skipped_expired` and discard
   e. If active: proceed to step 8

   **Do not halt the entire scan if one URL fails.** If `browser_navigate` errors (timeout, 403, etc.), mark as `skipped_expired` and continue with the next.

8. **For each verified new offer that passes filters**:
   a. Add to `pipeline.md` under the "Pendientes" section: `- [ ] {url} | {company} | {title}`
   b. Log in `scan-history.tsv`: `{url}\t{date}\t{query_name}\t{title}\t{company}\tadded`

9. **Offers filtered by title**: log in `scan-history.tsv` with status `skipped_title`
10. **Duplicate offers**: log with status `skipped_dup`
11. **Expired offers (Tier 3)**: log with status `skipped_expired`

## Title and company extraction from WebSearch results

WebSearch results come in the format: `"Job Title @ Company"` or `"Job Title | Company"` or `"Job Title — Company"`.

Extraction patterns per portal:
- **Ashby**: `"Senior AI PM (Remote) @ EverAI"` → title: `Senior AI PM`, company: `EverAI`
- **Greenhouse**: `"AI Engineer at Anthropic"` → title: `AI Engineer`, company: `Anthropic`
- **Lever**: `"Product Manager - AI @ Temporal"` → title: `Product Manager - AI`, company: `Temporal`

Generic regex: `(.+?)(?:\s*[@|—–-]\s*|\s+at\s+)(.+?)$`

## Private URLs

If a URL is found that is not publicly accessible:
1. Save the JD to `jds/{company}-{role-slug}.md`
2. Add to pipeline.md as: `- [ ] local:jds/{company}-{role-slug}.md | {company} | {title}`

## Scan History

`data/scan-history.tsv` tracks ALL URLs seen:

```
url	first_seen	portal	title	company	status
https://...	2026-02-10	Ashby — AI PM	PM AI	Acme	added
https://...	2026-02-10	Greenhouse — SA	Junior Dev	BigCo	skipped_title
https://...	2026-02-10	Ashby — AI PM	SA AI	OldCo	skipped_dup
https://...	2026-02-10	WebSearch — AI PM	PM AI	ClosedCo	skipped_expired
```

## Output summary

```
Portal Scan — {YYYY-MM-DD}
━━━━━━━━━━━━━━━━━━━━━━━━━━
Queries run: N
Offers found: N total
Filtered by title: N relevant
Duplicates: N (already evaluated or in pipeline)
Expired discarded: N (dead links, Tier 3)
New added to pipeline.md: N

  + {company} | {title} | {query_name}
  ...

→ Run /career-ops pipeline to evaluate the new offers.
```

## careers_url management

Every company in `tracked_companies` must have a `careers_url` — the direct URL to its jobs page. This avoids looking it up every time.

**RULE: Always use the company's corporate URL; fall back to the ATS endpoint only if the company has no dedicated corporate page.**

The `careers_url` must point to the company's own careers page whenever available. Many companies use Workday, Greenhouse, or Lever under the hood but expose job IDs only through their corporate domain. Using the direct ATS URL when a corporate page exists can cause false 410 errors because the job IDs do not match.

| ✅ Correct (corporate) | ❌ Wrong as first choice (direct ATS) |
|---|---|
| `https://careers.mastercard.com` | `https://mastercard.wd1.myworkdayjobs.com` |
| `https://openai.com/careers` | `https://job-boards.greenhouse.io/openai` |
| `https://stripe.com/jobs` | `https://jobs.lever.co/stripe` |

Fallback: if you only have the direct ATS URL, navigate first to the company's website and locate its corporate careers page. Use the direct ATS URL only if the company has no dedicated corporate page.

**Known patterns per platform:**
- **Ashby:** `https://jobs.ashbyhq.com/{slug}`
- **Greenhouse:** `https://job-boards.greenhouse.io/{slug}` or `https://job-boards.eu.greenhouse.io/{slug}`
- **Lever:** `https://jobs.lever.co/{slug}`
- **BambooHR:** list `https://{company}.bamboohr.com/careers/list`; detail `https://{company}.bamboohr.com/careers/{id}/detail`
- **Teamtailor:** `https://{company}.teamtailor.com/jobs`
- **Workday:** `https://{company}.{shard}.myworkdayjobs.com/{site}`
- **Custom:** The company's own URL (e.g., `https://openai.com/careers`)

**API/feed patterns per platform:**
- **Ashby API:** `https://jobs.ashbyhq.com/api/non-user-graphql?op=ApiJobBoardWithTeams`
- **BambooHR API:** list `https://{company}.bamboohr.com/careers/list`; detail `https://{company}.bamboohr.com/careers/{id}/detail` (`result.jobOpening`)
- **Lever API:** `https://api.lever.co/v0/postings/{company}?mode=json`
- **Teamtailor RSS:** `https://{company}.teamtailor.com/jobs.rss`
- **Workday API:** `https://{company}.{shard}.myworkdayjobs.com/wday/cxs/{company}/{site}/jobs`

**If `careers_url` does not exist** for a company:
1. Try the pattern of its known platform
2. If that fails, run a quick WebSearch: `"{company}" careers jobs`
3. Navigate with Playwright to confirm it works
4. **Save the found URL in portals.yml** for future scans

**If `careers_url` returns 404 or a redirect:**
1. Note it in the output summary
2. Try scan_query as fallback
3. Flag for manual update

## portals.yml maintenance

- **ALWAYS save `careers_url`** when adding a new company
- Add new queries as interesting portals or roles are discovered
- Disable queries with `enabled: false` if they generate too much noise
- Adjust filter keywords as target roles evolve
- Add companies to `tracked_companies` when you want to follow them closely
- Verify `careers_url` periodically — companies change ATS platforms
