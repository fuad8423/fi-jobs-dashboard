# FI Job Search — Reference Guide

## Overview
Multi-firm Fixed Income job scraper + dashboard. Scrapes career sites for senior FI roles (US-only, VP+), filters for relevance, and displays everything in a live dashboard.

**Local Dashboard URL:** `http://165.22.0.119:8765`
**Vercel Dashboard URL:** `https://fi-jobs-dashboard.vercel.app/`

---

## Running a Refresh (Recommended)

**Use MiMo v2.5 Pro via OpenRouter** — this workflow is 90% tool-calling and shell commands, not deep reasoning.

From the main OpenClaw session, spawn a sub-agent:
```
/spawn model=openrouter/xiaomi/mimo-v2.5-pro task="Read ~/pm_search_bot/REFRESH_PROMPT.txt and execute every step in order. Follow all critical rules. Report results at the end: total roles, dashboard roles, any errors, and what was deployed."
```

Or via the OpenClaw API:
```python
sessions_spawn(
    task="Read ~/pm_search_bot/REFRESH_PROMPT.txt and execute every step in order. Follow all critical rules. Report results at the end.",
    taskName="fi-jobs-refresh",
    model="openrouter/xiaomi/mimo-v2.5-pro",
    runTimeoutSeconds=3600
)
```

Typical runtime: ~14 minutes. Typical cost: < $0.10.

---

## Fit Scoring — Vector Similarity

Fit scores (0-100) measure how well each job matches Fuad's resume. **Always use vector similarity** — the old keyword-based `score_role()` is only a fallback.

### How it works
1. Extract resume text from `~/pm_search_bot/Fuad_Mahmood_Resume_InsurancePM.pdf`
2. Encode resume + each job's title+location using `sentence-transformers` (`all-MiniLM-L6-v2`)
3. Compute cosine similarity between resume vector and each job vector
4. Min-max scale to 0-100

### Running fit scoring
```bash
cd ~/pm_search_bot && source venv/bin/activate
python3 << 'EOF'
import json
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np
import pdfplumber

model = SentenceTransformer('all-MiniLM-L6-v2')
with pdfplumber.open('Fuad_Mahmood_Resume_InsurancePM.pdf') as pdf:
    resume = ' '.join(p.extract_text() or '' for p in pdf.pages)
resume_vec = model.encode([resume])

with open('fi_jobs_raw.json') as f:
    jobs = json.load(f)
titles = [f"{j.get('title','')} {j.get('location','')}" for j in jobs]
vecs = model.encode(titles)
sims = cosine_similarity(resume_vec, vecs)[0]
lo, hi = sims.min(), sims.max()
for j, s in zip(jobs, sims):
    j['fit'] = round((s - lo) / (hi - lo) * 100, 1) if hi > lo else 50.0
json.dump(jobs, open('fi_jobs_raw.json','w'), indent=2)
print(f'Updated {len(jobs)} jobs. Range: {lo:.3f} – {hi:.3f}')
EOF
```

### Preserving fit scores during rebuild
`generate_dashboard_data()` in `fi_job_scraper.py` preserves existing fit scores:
```python
existing_fit = j.get("fit", 0)
fit = existing_fit if existing_fit and existing_fit > 0 else score_role(title, firm, seniority)
```
Always run fit scoring **before** `--refresh-md`. The rebuild will keep whatever scores are in `fi_jobs_raw.json`.

---

## Last Refresh Summary (2026-07-20)

**Status:** ✅ Refresh completed 2026‑07‑20
**Total roles (raw):** 295
**Dashboard (filtered ≥$200k, no Analysts):** 216 roles
**Firms (raw):** 21
**Firms (dashboard):** 17
**Model:** openrouter/xiaomi/mimo-v2.5-pro
**Deployed to Vercel:** ✅

**What changed from 2026-06-26:**
- Full re-scrape of all firms (Goldman Sachs, 12 Workday firms, iCIMS)
- 66 new roles merged, 98 stale roles pruned
- 14 recent roles had no seniority — fixed and rebuilt (see Known Issues)
- Fit scores recalculated via vector similarity (325 jobs, range 0.053–0.646)
- Added Step 6b (seniority fix) and Step 13 (dashboard verification) to refresh workflow
- Removed gemini-2.5-flash references; using mimo-v2.5-pro for sub-agents

**Previous refresh (2026-06-22):**
- 347 raw roles, 269 dashboard roles
- Goldman Sachs: 46 FI roles
- 48 manually curated roles
- 36 stale roles removed, 23 broken links pruned

**Issues & Fixes (2026‑05-28):**

### 1. Broken Regex Exclude Filter
The `exclude_keywords` list contains regex‑special characters (`C++`) that were not escaped, causing regex compilation errors and incorrect matching (the pattern `C++` as regex is invalid). This caused many legitimate FI roles to be incorrectly filtered out, resulting in far fewer roles per firm than the manually curated backup.

**Fix:** Add regex escaping for all literal keyword strings before joining them with `|`. Modified `fi_job_scraper.py`:
- `escape_re = lambda s: re.escape(s)`
- Apply to `fi_keywords`, `exclude_keywords`, `us_locations`, `exclude_locations`, `seniority_keywords` across all scrapers (`scrape_workday`, `scrape_oracle_ats`, `scrape_higher_gs`).
- Also fixed `workday_api.py` to escape literal FI keywords (keeping extra regex patterns like `capital.market` unchanged).

### 2. Significant Drop in Workday Firm Results After Fix
After fixing regex, the number of Apollo roles increased from 10 to 29, but still fell far short of the backup’s 40 roles. The backup contained manually curated roles that were no longer posted on the live site (expired/removed jobs) and some that had been added manually during earlier scrapes before the filter was stricter.

**Decision:** Prune stale roles from backup. Updated refresh workflow to compare backup titles against fresh API results and drop any titles that do not appear in the fresh scrape.

**Result:** Raw roles dropped from 345 → 182, dashboard roles from 311 → 152. This reflects the true current job market for those firms (removing stale postings).

### 3. Workday API Pagination Caps (Guardian Life, Voya)
Guardian Life and Voya Financial have Workday API pagination caps (40 results). Use `searchText` keyword queries to work around the limit (split by department/keyword).

### 4. iCIMS 403 (Principal) — Fixed 2026‑06‑05
Principal blocks Playwright with 403. **Fix:** Use Firecrawl API (`scrape_firecrawl()` in `fi_job_scraper.py`). Firecrawl bypasses the WAF. API key in `~/.openclaw/credentials/firecrawl.json`. Now returns 11 FI roles (6 on dashboard).

### 5. Link Validation Step
After merging fresh scrapes and pruning stale titles, validate all remaining links (HTTP HEAD) and prune any that return 404/403 or fail to resolve. Use `validate_links.py` script (added 2026‑05‑28) to automate this step in the refresh workflow.

### 6. Next Steps
- Add periodic manual curation pass for firms with big gaps (Apollo, PIMCO, Capital Group, Guardian Life, Voya, Pacific Life, New York Life).
- Consider expanding `fi_keywords` to include terms like `ALM`, `CLO`, `Syndicate`, `Liquid Credit`, `Multi-Credit` (some already in `extra_fi` patterns).
- Review `exclude_keywords` list to ensure it does not inadvertently filter FI roles (e.g., `Securities Settlements` is appropriate, `Product Control` maybe not).

### 7. Scraper Merge Logic Bug — Fixed 2026‑06‑03

**Problem:** The `--firm` merge logic in `fi_job_scraper.py` dropped ALL existing roles for any firm that was re-scraped, even if the fresh scrape returned fewer roles. This caused data loss when a firm's API returned fewer results than the backup.

**Root cause:** The merge used `firm not in scraped_firms` as the keep condition, which meant any firm that appeared in the fresh scrape had ALL its existing roles discarded:
```python
# BROKEN: drops all existing roles for re-scraped firms
if j.get("firm", "") not in {j2["firm"] for j2 in all_jobs}:
    all_jobs.append(j)
```

**Fix:** Changed merge to use `(firm, title)` as dedup key. For re-scraped firms, existing roles whose title wasn't found in the fresh scrape are preserved:
```python
# FIXED: preserves existing roles not found in fresh scrape
scraped_firms = {j2["firm"] for j2 in all_jobs}
fresh_keys = {(j.get("firm",""), j.get("title","")) for j in all_jobs}
for key, j in existing.items():
    firm = j.get("firm", "")
    title = j.get("title", "")
    if firm in scraped_firms:
        if (firm, title) not in fresh_keys:
            all_jobs.append(j)
    else:
        all_jobs.append(j)
```

**Impact:** Backup data preserved for firms that return 0 or fewer roles on fresh scrape (J.P. Morgan, Ares, Athene, Guggenheim returned 0 this run).

**Note:** The stale-prune step (Step 7 in refresh workflow) should still be run separately for firms with ≥5 fresh roles to remove genuinely expired postings.

### 8. Sub-Agent Timeout During Full Refresh — 2026‑06‑26

**Problem:** The sub-agent timed out after ~16 minutes while working through the 12-step refresh workflow. It completed Steps 1–6 (backup, GS scrape, Workday firms, iCIMS, partial posted dates, partial fit scoring) but did not finish Steps 7–12 (pruning, link validation, rebuild, deploy).

**Root cause:** The workflow is long (~14 min ideal) and the sub-agent hit the default session timeout before completing all steps. The sub-agent also corrupted `fi_jobs_raw.json` by writing a partial numpy float32 value that wasn't JSON-serializable.

**Fix (if it happens):**
1. Check what the sub-agent completed: `ls -la /tmp/fi_jobs_*.json /tmp/gs_fi_jobs.json`
2. Restore from backup if JSON is corrupted: `cp /tmp/fi_jobs_raw_backup.json fi_jobs_raw.json`
3. Continue manually from the last completed step (usually Step 8 — fit scoring)
4. Fit scoring requires converting numpy float32 to Python float: `round(float(val), 1)`
5. Complete remaining steps (prune → fit scoring → link validation → rebuild → deploy)

**Prevention:** Consider increasing `runTimeoutSeconds` to 2400+ or breaking into two sub-agents (scrape+merge, then score+deploy).

**Known issues (updated):**
- ⚠️ **GIC**: Scraper type `gic_careers` not implemented (issue #20) — using 4 curated roles from backup
- ⚠️ **Millennium**: Scraper type `millennium` not implemented (issue #20) — using 3 curated roles from backup
- ✅ **Principal**: ~~iCIMS ATS blocks all access~~ — Fixed 2026‑06‑05 via Firecrawl API. 11 FI roles, 6 on dashboard.
- ⚠️ **GS**: 0% posted dates (`higher.gs.com` does not expose dates — cannot be fixed)
- ⚠️ **Workday pagination cap**: Guardian Life and Voya Financial return `total: N` but only serve 40 jobs. Workaround: use keyword `searchText` queries (see issue #23)
- ⚠️ **J.P. Morgan / Ares / Athene / Guggenheim**: Workday API returned 0 FI roles on 2026‑06‑03 run. AI filter may need tuning. Backup data preserved by merge fix.

---

## Current Firms (309 raw roles — 23 firms)

| Firm | Type | Raw Roles | FI Filtered | Posted Dates | Status |
|------|------|-----------|-------------|-------------|--------|
| Apollo | Workday wd5 | 40 | 40 | ✅ API | ✅ |
| Ares | Workday wd1 | 19 | 19 | ✅ API | ✅ |
| Athene | Workday wd5 | 14 | 14 | ✅ API | ✅ manual curation |
| Blackstone | Workday wd1 | 33 | 33 | ✅ API | ✅ |
| Brookfield | Workday wd5 | 10 | 10 | ✅ API | ✅ |
| Capital Group | Workday wd1 | 13 | 13 | ✅ API | ✅ fixed endpoint |
| GIC | gic_careers | 4 | 4 | ❌ | ⚠️ not implemented |
| Franklin Templeton | Workday wd5 | 18 | 18 | ✅ API | ✅ keyword search (pagination cap) |
| Goldman Sachs | higher_gs | 47 | 31 | ❌ | ✅ gs_scrape.py |
| Guardian Life | Workday wd5 | 16 | 16 | ✅ API | ✅ pagination workaround |
| Guggenheim | Workday wd5 | 12 | 12 | ✅ API | ✅ manual curation |
| Invesco | Workday wd1 | 16 | 16 | ✅ API | ✅ |
| J.P. Morgan | Oracle ATS | 17 | 17 | ✅ visible | ✅ |
| Millennium | millennium | 3 | 3 | ❌ | ⚠️ not implemented |
| New York Life | Eightfold | 14 | 14 | ❌ | ✅ Playwright interception |
| Nuveen | TIAA | 18 | 18 | ✅ JSON-LD | ✅ expanded keywords + seniority fix
| Neuberger Berman | Workday wd1 | 102 | 4 | ✅ API | ✅ New firm
| PIMCO | Workday wd1 | 31 | 31 | ✅ API | ✅ fixed endpoint |
| Pacific Life | Workday wd1 | 21 | 21 | ✅ API | ✅ fixed endpoint |
| Principal | Firecrawl | 11 | 6 | ✅ scrape | ✅ Firecrawl API |
| TCW | iCIMS | 11 | 11 | ❌ | ✅ web_fetch (in_iframe=1) |
| Voya Financial | Workday wd5 | 18 | 18 | ✅ API | ✅ pagination workaround |
| Wellington Management | Workday wd5 | 134 | 6 | ✅ API | ✅ pagination workaround (keyword search) |

---

## Architecture

```
career_sites.json          ← Config: add/edit firms here (firm name, type, URL, enabled)
       ↓
fi_job_scraper.py          ← Main scraper script (--firm "Name" or full scrape)
       ↓
fi_jobs_raw.json           ← Raw scraped data (cached) — source of truth (unfiltered)
       ↓
jpm_fi_jobs_links.md       ← Human-readable markdown (auto-generated, --refresh-md)
       ↓
generate_dashboard_data()  ← Applies comp/seniority filter to data.json
       ↓
~/dashboard/jpm-fi-jobs/data.json  ← Dashboard data (filtered: no Analysts, min $200k)
       ↓
index.html                 ← Live dashboard (port 8765 or Vercel)
```

---

## Quick Commands

```bash
cd ~/pm_search_bot && source venv/bin/activate

# Full scrape — all enabled firms
python3 fi_job_scraper.py

# Scrape one firm only (⚠️ backs up fi_jobs_raw.json first)
python3 fi_job_scraper.py --firm "Blackstone"

# Skip scraping, just rebuild markdown + dashboard from cached data
python3 fi_job_scraper.py --refresh-md

# Dump all job titles from a firm (for manual curation)
python3 -u dump_titles.py "Apollo"

# Collect direct apply links for curated roles missing links
python3 -u collect_links.py

# Scrape Goldman Sachs (dedicated script — avoids EPIPE crash)
python3 -u gs_scrape.py
```

---

## Workday API Reference

All Workday sites expose a JSON API — no Playwright needed.

### Endpoint Pattern
```
POST https://{tenant}.wd{N}.myworkdayjobs.com/wday/cxs/{tenant}/{site}/jobs
Content-Type: application/json
{"appliedFacets": {}, "limit": 20, "offset": 0, "searchText": ""}
```

### Detail Page (for posted dates)
```
GET https://{tenant}.wd{N}.myworkdayjobs.com/wday/cxs/{tenant}/{site}/job/{externalPath}
```
The detail response has `jobPostingInfo.postedOn` — typically relative ("Posted 12 Days Ago"). Convert to absolute `MM/DD/YYYY` at scrape time.

### Current Workday Tenant Mappings

| Firm | Tenant | Site | API Endpoint |
|------|--------|------|-------------|
| Apollo | athene | Apollo_Careers | `athene.wd5.myworkdayjobs.com/wday/cxs/athene/Apollo_Careers/jobs` |
| Ares | aresmgmt | External | `aresmgmt.wd1.myworkdayjobs.com/wday/cxs/aresmgmt/External/jobs` |
| Athene | athene | athene_careers | `athene.wd5.myworkdayjobs.com/wday/cxs/athene/athene_careers/jobs` |
| Blackstone | blackstone | Blackstone_Careers | `blackstone.wd1.myworkdayjobs.com/wday/cxs/blackstone/Blackstone_Careers/jobs` |
| Brookfield | brookfield | brookfield | `brookfield.wd5.myworkdayjobs.com/wday/cxs/brookfield/brookfield/jobs` |
| Capital Group | capgroup | capitalgroupcareers | `capgroup.wd1.myworkdayjobs.com/wday/cxs/capgroup/capitalgroupcareers/jobs` |
| Franklin Templeton | franklintempleton | Primary-External-1 | `franklintempleton.wd5.myworkdayjobs.com/wday/cxs/franklintempleton/Primary-External-1/jobs` |
| Guardian Life | guardianlife | Guardian-Life-Careers | `guardianlife.wd5.myworkdayjobs.com/wday/cxs/guardianlife/Guardian-Life-Careers/jobs` |
| Guggenheim | guggenheiminvestment | External | `guggenheiminvestment.wd5.myworkdayjobs.com/wday/cxs/guggenheiminvestment/External/jobs` |
| Invesco | invesco | IVZ | `invesco.wd1.myworkdayjobs.com/wday/cxs/invesco/IVZ/jobs` |
| PIMCO | pimco | PIMCO-Careers | `pimco.wd1.myworkdayjobs.com/wday/cxs/pimco/PIMCO-Careers/jobs` |
| Pacific Life | pacificlife | PacificLifeCareers | `pacificlife.wd1.myworkdayjobs.com/wday/cxs/pacificlife/PacificLifeCareers/jobs` |
| Voya Financial | godirect | voya_jobs | `godirect.wd5.myworkdayjobs.com/wday/cxs/godirect/voya_jobs/jobs` |
| Wellington Management | wellington | External | `wellington.wd5.myworkdayjobs.com/wday/cxs/wellington/External/jobs` |

### Pagination Cap Warning
Guardian Life, Voya Financial, and Wellington Management: API returns `total: N` but pagination stops at 40 jobs. The `offset` param returns empty `jobPostings[]` after 40. **Workaround:** Use keyword `searchText` queries to find all jobs across multiple searches.

---

## iCIMS ATS Notes

Principal and TCW use iCIMS. iCIMS has no public REST API.

### What works
- **TCW**: `web_fetch` works — append `?in_iframe=1` to listing URL. Parsing HTML titles and links.
- **Principal**: Firecrawl API bypasses the WAF. No other method works (Playwright, curl, stealth all return 403).

### How to scrape TCW (proven 2026-05-22)
```bash
curl -s 'https://careers-tcw.icims.com/jobs/search?ss=1&searchKeyword=&searchCategory=&searchPostedDate=&in_iframe=1'
```
or with web_fetch:
```
https://careers-tcw.icims.com/jobs/search?ss=1&searchKeyword=&searchCategory=&searchPostedDate=&in_iframe=1
```
Each job is a markdown bullet with title and link. Extract all titles, filter for FI, add to `fi_jobs_raw.json`.

### How to scrape Principal (proven 2026-06-05)
Principal's iCIMS site blocks all automated access (Playwright, curl, stealth mode all return 403). The **only** method that works is the Firecrawl API.

```bash
cd ~/pm_search_bot && source venv/bin/activate

# Scrape via the scraper (uses Firecrawl automatically)
python3 fi_job_scraper.py --firm "Principal"

# Or manual scrape with Firecrawl directly:
python3 << 'EOF'
from firecrawl import FirecrawlApp
import re, json

app = FirecrawlApp(api_key="<your-firecrawl-api-key>")
all_jobs = {}

for page in range(1, 15):
    url = f"https://careers.principal.com/careers-home/jobs?page={page}"
    result = app.scrape_url(url)
    md = result.markdown or ''
    blocks = re.split(r'\[Read More\]', md)
    page_count = 0
    for block in blocks:
        m = re.search(r'\[([^\]]+)\]\(https://careers\.principal\.com/careers-home/jobs/(\d+)', block)
        if not m or m.group(1) == 'Read More': continue
        req_id = m.group(2)
        if req_id in all_jobs: continue
        loc = re.search(r'Location\s+([^\n]+)', block)
        cat = re.search(r'Category\s+([^\n]+)', block)
        posted = re.search(r'Posted\s+(\d{2}/\d{2}/\d{4})', block)
        all_jobs[req_id] = {
            'title': m.group(1), 'req_id': req_id,
            'location': loc.group(1).strip() if loc else 'Multiple',
            'category': cat.group(1).strip() if cat else '',
            'posted': posted.group(1) if posted else '',
            'link': f'https://careers.principal.com/careers-home/jobs/{req_id}'
        }
        page_count += 1
    if page_count == 0: break

# Filter for FI roles (Investments category or FI keywords)
fi = [j for j in all_jobs.values() if 'investments' in j['category'].lower() \
      or any(k in j['title'].lower() for k in ['fixed income','credit','portfolio','trader','derivatives','director'])]
print(json.dumps(fi, indent=2))
EOF
```

**Key details:**
- API key stored in `~/.openclaw/credentials/firecrawl.json`
- Scraper type in `career_sites.json`: `"icims"` or `"firecrawl"`
- Site returns 10 jobs per page, ~81 total jobs
- Keyword filtering is client-side JS — scrape all pages, filter locally
- Buy-side seniority convention applies: "Analyst" → Senior, "Associate" → Vice President

### Alternative methods (untested on these sites)
- `web_fetch` without `in_iframe=1` returns empty shell (JS-rendered SPA)

---

## TIAA/Nuveen Careers Notes (proven 2026-06-05)

Nuveen uses TIAA's careers site (`careers.tiaa.org`). Type: `tiaa_careers`. ~143 total jobs, 10 per page, paginated via `/jobs/page/N`.

### How it works
1. Playwright loads each page, extracts all `<a href="/job/">` links
2. `fi_keywords` filter matches titles client-side
3. Seniority is assigned from title keywords (see table below)

### Key gotchas

**1. Seniority must be assigned from the title.** The TIAA site does not expose seniority in the HTML. Without explicit tagging, all roles default to `seniority: "—"` → estimated comp $150k → filtered out by the $200k dashboard minimum.

**Fix in `fi_job_scraper.py` — `scrape_tiaa_careers()`:**
```python
# Assign seniority from title
t_lower = j["title"].lower()
if "managing director" in t_lower or t_lower.startswith("md "):
    seniority = "MD"
elif "executive director" in t_lower:
    seniority = "ED"
elif "head of" in t_lower or "head -" in t_lower:
    seniority = "Director"
elif "portfolio manager" in t_lower or "portfolio strategist" in t_lower or "portfolio management" in t_lower:
    seniority = "Portfolio Manager"
elif "svp" in t_lower or "senior vice president" in t_lower:
    seniority = "SVP"
elif "director" in t_lower:
    seniority = "Director"
elif "vice president" in t_lower or t_lower.startswith("vp,") or t_lower.startswith("vp "):
    seniority = "Vice President"
elif "avp" in t_lower:
    seniority = "AVP"
elif "associate" in t_lower:
    seniority = "Vice President"  # buy-side convention
elif "analyst" in t_lower:
    seniority = "Senior"  # buy-side convention
elif re.search(r'\bsr\.?\b', t_lower) or "senior" in t_lower:
    seniority = "Senior"
elif "lead" in t_lower:
    seniority = "Lead"
elif "manager" in t_lower:
    seniority = "Manager"
elif "quant" in t_lower:
    seniority = "Senior"
else:
    seniority = "—"
```

**2. Expand `fi_keywords` in `career_sites.json` for better coverage.**
Nuveen uses terms like "Capital Markets", "ETF", "Treasury" that aren't caught by the default FI keywords. Add these to `filters.fi_keywords`:
```
"capital markets", "ETF", "treasury", "investment grade", "high yield",
"CLO", "syndicate", "liquid credit", "multi-credit", "ALM",
"yield", "duration", "leverage", "mezzanine", "direct lending", "ABS", "taxable fixed"
```

**3. Edge cases — titles without standard keywords.**
Some titles don't match any seniority keyword:
- "Credit Risk Methodology and Oversight" — no seniority keyword → manually tag as `Senior`
- "Taxable Fixed Income Sr Research Analyst - ABS" — has "Sr" but regex may miss it in compound titles → verify after scrape

For these, either:
- Add the role manually to `fi_jobs_raw.json` with correct seniority, OR
- Expand the seniority keyword list in the scraper

### Scrape command
```bash
cd ~/pm_search_bot && source venv/bin/activate
python3 fi_job_scraper.py --firm "Nuveen"
```

### Expected result
- ~143 total TIAA jobs scraped
- ~17-18 FI-filtered roles (with expanded keywords)
- All roles should have seniority assigned → all appear on dashboard

---

## Franklin Templeton Careers Notes (proven 2026-06-07)

Franklin Templeton uses Workday wd5 (`franklintempleton.wd5.myworkdayjobs.com`). 174 total jobs, 18 FI roles. API endpoint: `Primary-External-1`.

### Pagination Cap (40 jobs)
Like Guardian Life and Voya, the Workday API stops returning results after offset=40. **Workaround:** Use keyword `searchText` queries to find all jobs across multiple searches.

### How to scrape
```bash
cd ~/pm_search_bot && source venv/bin/activate

# Via the scraper (uses Workday API)
python3 fi_job_scraper.py --firm "Franklin Templeton"

# Or manual keyword search (bypasses 40-cap):
python3 << 'EOF'
import requests, json
api = 'https://franklintempleton.wd5.myworkdayjobs.com/wday/cxs/franklintempleton/Primary-External-1/jobs'
headers = {'Content-Type': 'application/json'}
all_jobs = {}
for term in ['fixed income','credit','portfolio','bond','trader','derivatives','municipal','structured','mortgage','MBS','CLO','treasury','capital markets','high yield','investment grade','ABS','syndicate','quantitative','risk','attribution','insurance','liquidity']:
    r = requests.post(api, headers=headers, json={'appliedFacets':{}, 'limit':20, 'offset':0, 'searchText':term}, timeout=20)
    for j in r.json().get('jobPostings', []):
        all_jobs[j['externalPath']] = j
print(f'Total unique: {len(all_jobs)}')
EOF
```

### Key details
- 18 FI roles found via keyword search (all dashboard-eligible)
- All roles have posted dates via detail API
- Includes Putnam Investments and Western Asset (FT subsidiaries)
- Seniority is assigned from title keywords (same logic as TIAA/Nuveen)
- FI roles include: Portfolio Managers, Quant PMs, Traders, Risk Analysts, Research Analysts, Quant Developers

### Expected result
- ~174 total jobs on site
- ~18 FI-filtered roles (after keyword + location filtering)
- All 18 should appear on dashboard (all have seniority ≥ Senior)

---

## Applying to Roles

Resume: `~/pm_search_bot/Fuad_Mahmood_Resume_InsurancePM.pdf`
Credentials: `~/.openclaw/credentials/franklin_templeton.json`

### Workflow (proven 2026-06-07)

1. **Find the role** in `fi_jobs_raw.json` or the dashboard
2. **Get the apply link** from the role's `link` field
3. **Open the link** in a headless browser (Playwright)
4. **Login or create account** — Workday sites require authentication
5. **Click Apply** — most Workday sites have an `<a>` link (not a button) for Apply
6. **Choose "Autofill with Resume"** — upload the resume PDF
7. **Click Continue** → proceeds to Step 2 (My Information)
8. **Fill required fields** — the autofill populates name, email, phone from the resume
9. **Missing fields to fill manually:**
   - Address Line 1, Postal Code
   - "Have you ever been employed here?" → No
   - "How Did You Hear About Us?" → **Workday multiselect widget — cannot be automated** (see below)
10. **Save and Continue** through remaining steps (Experience, Questions, Disclosures, Review)
11. **Mark as applied** in `fi_jobs_raw.json`: add `"applied": true, "applied_date": "YYYY-MM-DD"`
12. **Refresh dashboard**: `python3 fi_job_scraper.py --refresh-md` → deploy

### Dashboard Applied Tracking

The dashboard has an **Applied** column and filter:
- Roles with `"applied": true` show a green ✅ with the date
- Filter dropdown: All Status / Applied / Not Applied
- Header shows total applied count

To mark a role as applied:
```python
import json
with open('fi_jobs_raw.json') as f:
    raw = json.load(f)
for j in raw:
    if j['firm'] == 'Firm Name' and 'Role Title' in j['title']:
        j['applied'] = True
        j['applied_date'] = '2026-06-07'
with open('fi_jobs_raw.json', 'w') as f:
    json.dump(raw, f, indent=2)
```

Then refresh:
```bash
cd ~/pm_search_bot && source venv/bin/activate
python3 fi_job_scraper.py --refresh-md
cd ~/dashboard/jpm-fi-jobs && git add data.json && git commit -m "Mark applied" && git push
VERCEL_TOKEN=$(python3 -c "import json; print(json.load(open('/root/.openclaw/credentials/vercel.json'))['token'])")
vercel --prod --yes --token "$VERCEL_TOKEN"
```

### Workday "How Did You Hear About Us?" Limitation

The multiselect widget (`data-uxi-widget-type="multiselect"`) cannot be automated:
- Click events don't register on `[role="option"]` elements
- Keyboard navigation (ArrowDown + Enter) highlights but doesn't select
- React `onChange` dispatch doesn't update widget state
- Mouse event dispatching (mousedown/click) on options fails

**Workaround:** The applicant must manually select this field in the browser. Everything else can be autofilled.

---

### Greenhouse Application Notes (proven 2026-06-07)

Greenhouse job boards (`job-boards.greenhouse.io`) use invisible reCAPTCHA v3 that scores bot behavior. Standard Playwright headless fails silently — form fills, submits, but redirects back without submission.

**What works:**
- **nodriver** (v0.50.3) — undetected Chrome, passes reCAPTCHA, form submits with zero errors
- File upload: `element.send_file(path)` (NOT `send_keys`)
- React-Select dropdowns: JS `focus()/click()` + keyboard dispatch events
- Location field: `id="candidate-location"` with `role="combobox"` (not `aria-label="Location (City)"`)

**What does NOT work (do not retry):**

| Approach | Why it fails |
|----------|-------------|
| **Playwright headless** | reCAPTCHA v3 invisible bot detection — zero network responses after form submit, redirects back silently |
| **Playwright with stealth plugin** | Still detected — reCAPTCHA uses behavioral analysis (mouse movements, timing), not just navigator flags |
| **`send_keys()` on file input** | Does not attach files — nodriver requires `send_file(path)` for `<input type="file">` |
| **`send_keys()` on React-Select parent div** | Keystrokes go to the `<div>`, not the hidden `<input role="combobox">` — must target the input directly |
| **`aria-label="Location (City)"` selector** | Greenhouse uses `id="candidate-location"` instead — the aria-label varies by board |
| **`click()` on React-Select options** | React-Select renders options as `<div>` in a portal — clicks don't register on them. Use keyboard Enter instead |
| **JS `nativeSetter` for React-Select** | Sets the DOM value but doesn't update React state — dropdown appears empty on submit. Must use focus+type+Enter |
| **Tab navigation into code inputs** | MyGreenhouse security code inputs are 8 separate `<input>` fields — tab doesn't advance between them reliably. Use JS to fill all at once |
| **`playwright-stealth`, `patchright`** | Both still detected by reCAPTCHA v3 — only nodriver passes |

**MyGreenhouse login gotchas:**
- Passwordless only — sends 8-character email code (valid 10 minutes)
- Each new browser session requesting a code invalidates all previous codes
- JS `nativeSetter` fills code inputs visually but doesn't trigger React state — submit returns "Invalid"
- Code file polling pattern: background process polls `/tmp/gh_code.txt` for user-provided code

**Recommended Greenhouse apply script:** `~/pm_search_bot/apply_greenhouse.py` (nodriver-based)

---

## Full Refresh Workflow — All Firms

### Step 1: Backup
```bash
cd ~/pm_search_bot
cp fi_jobs_raw.json fi_jobs_raw.json.bak.$(date +%Y%m%d)
cp fi_jobs_raw.json /tmp/fi_jobs_raw_backup.json
```

### Step 2: Scrape Goldman Sachs (dedicated script)
```bash
timeout 600 python3 -u gs_scrape.py
# Output: /tmp/gs_fi_jobs.json (~32 FI roles from ~741 total links)
```

### Step 3: Scrape Workday firms (REST API — one at a time)
**⚠️ Do NOT run full `python3 fi_job_scraper.py` — it overwrites `fi_jobs_raw.json`.**

```bash
cd ~/pm_search_bot && source venv/bin/activate

# Save GS before --firm overwrites it
cp fi_jobs_raw.json /tmp/fi_jobs_with_gs.json

for firm in "J.P. Morgan" "Apollo" "Blackstone" "PIMCO" "Capital Group" \
            "Pacific Life" "Guardian Life" "Nuveen" "Brookfield" \
            "Invesco" "Voya Financial" "New York Life" "Neuberger Berman" "Wellington Management"; do
  echo "--- Scraping $firm ---"
  timeout 120 python3 -u fi_job_scraper.py --firm "$firm" 2>&1 | tail -5
  cp fi_jobs_raw.json "/tmp/fi_jobs_$(echo $firm | tr ' ' '_').json"
done
```

### Step 4: Manual curation — dump titles and filter
```bash
cd ~/pm_search_bot && source venv/bin/activate

# Dump all titles from each firm
for firm in Apollo Ares Athene Blackstone Brookfield "Capital Group" GIC \
            "Goldman Sachs" "Guardian Life" Guggenheim Invesco "J.P. Morgan" \
            Millennium "New York Life" Nuveen PIMCO "Pacific Life" Principal TCW \
            "Voya Financial"; do
  echo "=== $firm ==="
  python3 -u dump_titles.py "$firm"
done
```

Review output for FI-relevant titles the auto-filter missed. Add manually to `fi_jobs_raw.json` with proper seniority (see Seniority Rules below).

### Step 5: Get posted dates from Workday detail API
```python
# For each role with a Workday link, hit the detail endpoint:
import requests, json, re
from datetime import datetime, timedelta

def relative_to_absolute(s):
    m = re.match(r'(?:Posted\s+)?(?:~?\s*)(\d+)(\+?)\s*(Day|Week|Month)s?\s+Ago', s, re.IGNORECASE)
    if m:
        n, unit = int(m.group(1)), m.group(3).lower()
        if unit == 'day': dt = datetime.now() - timedelta(days=n)
        elif unit == 'week': dt = datetime.now() - timedelta(weeks=n)
        else: dt = datetime.now() - timedelta(days=n*30)
        return dt.strftime('%m/%d/%Y')
    return s

# Example: Apollo detail page
api = 'https://athene.wd5.myworkdayjobs.com/wday/cxs/athene/Apollo_Careers/job/{externalPath}'
r = requests.get(api, headers={'Content-Type':'application/json'}, timeout=20)
posted = r.json().get('jobPostingInfo', {}).get('postedOn', '')
print(relative_to_absolute(posted))  # "05/10/2026"
```

### Step 5b: Fix location values

**This is now automatic.** `clean_locations()` runs inside `generate_dashboard_data()` before every dashboard publish. It:
- Strips "USA" / "United States of America" suffixes
- Resolves "N Locations" placeholders via Workday detail API (with manual fallbacks for non-Workday firms)
- Abbreviates full state names (New York→NY, Texas→TX, etc.)
- Normalizes to "City, ST" format

No manual intervention needed unless new non-Workday firms with "N Locations" are added — add them to `MANUAL_LOC` in `clean_locations()`.

**Check and fix:**
```bash
cd ~/pm_search_bot && source venv/bin/activate

# Dump all locations to spot issues
python3 -c "
import json
with open('fi_jobs_raw.json') as f:
    raw = json.load(f)
for j in raw:
    loc = j.get('location','')
    if 'Locations' in loc or 'United States of America' in loc:
        print(f'{j[\"firm\"]:25s} | {loc:50s} | {j[\"title\"]}')
"
```

**For "N Locations" entries:** Hit the Workday detail API to get the primary location:
```python
import requests, json
api_base = "https://{tenant}.wd{N}.myworkdayjobs.com/wday/cxs/{tenant}/{site}"
headers = {'Content-Type': 'application/json', 'Accept': 'application/json'}

# Extract path from role link
path = role['link'].replace('https://{tenant}.wd{N}.myworkdayjobs.com/en-US/{site}', '')
r = requests.get(f"{api_base}{path}", headers=headers, timeout=20)
info = r.json().get('jobPostingInfo', {})
primary_location = info.get('location', '')        # e.g. "New York, New York, United States of America"
additional = info.get('additionalLocations', [])    # e.g. ["Boston, ...", "Chicago, ..."]
```

**Rules:**
- Use the **primary location** from the detail API for the `location` field
- Strip `", United States of America"` suffix → keep `"City, State"` only
- **Remove international roles** (Edinburgh, Singapore, London, etc.) — they don't belong on the dashboard
- **Tag as `"multiple"`** if the role is genuinely multi-location and you want to note that

**Quick fix script:**
```python
import json, requests, re

with open('fi_jobs_raw.json') as f:
    raw = json.load(f)

for j in raw:
    loc = j.get('location', '')
    if 'Locations' not in loc and 'United States of America' not in loc:
        continue
    # Hit detail API to resolve
    path = j['link'].replace(f"https://{host}/en-US/{site}", "")
    r = requests.get(f"{api_base}{path}", headers=headers, timeout=20)
    info = r.json().get('jobPostingInfo', {})
    primary = info.get('location', '')
    if any(x in primary for x in ['United Kingdom','Singapore','Hong Kong','India']):
        j['_remove'] = True
        continue
    j['location'] = primary.replace(', United States of America', '')

raw = [j for j in raw if not j.get('_remove')]
json.dump(raw, open('fi_jobs_raw.json', 'w'), indent=2)
```

### Step 6: Merge all firm data into fi_jobs_raw.json
```python
import json

# Start with backup
raw = json.load(open('/tmp/fi_jobs_raw_backup.json'))

# Merge all firm temp files
for fname in ['fi_jobs_with_gs.json', 'fi_jobs_J.P._Morgan.json', 'fi_jobs_Apollo.json', ...]:
    firm_data = json.load(open(f'/tmp/{fname}'))
    existing = {(j['firm'], j['title']) for j in raw}
    for j in firm_data:
        if (j['firm'], j['title']) not in existing:
            raw.append(j)
            existing.add((j['firm'], j['title']))

json.dump(raw, open('fi_jobs_raw.json', 'w'), indent=2)
print(f'Merged: {len(raw)} total roles')
```

### Step 7: Rebuild and deploy
```bash
cd ~/pm_search_bot && source venv/bin/activate
python3 fi_job_scraper.py --refresh-md

cd ~/dashboard/jpm-fi-jobs
git add data.json
git commit -m "Full refresh — $(date +%Y-%m-%d)"
git push origin main
vercel --prod --yes --token "$(python3 -c "import json; print(json.load(open("/root/.openclaw/credentials/vercel.json"))[\"token\"])")"
```

### Step 8: Verify dashboard shows recent roles
After deploy, always verify that recently posted roles actually appear in the dashboard. New roles from Workday APIs often lack seniority, causing them to be silently dropped by the comp filter.

```bash
cd ~/pm_search_bot && python3 -c "
import json
from datetime import datetime, timedelta
with open('fi_jobs_raw.json') as f: raw = json.load(f)
with open('../dashboard/jpm-fi-jobs/data.json') as f: dash = json.load(f)
dash_titles = {r['title'] for r in dash.get('roles', [])}
cutoff = datetime.now() - timedelta(days=14)
missing = [j for j in raw if (lambda d: d and datetime.strptime(d, '%m/%d/%Y') >= cutoff)(j.get('posted','')) and j.get('title','') not in dash_titles]
if missing:
    print(f'⚠️ {len(missing)} recent roles MISSING — assign seniority, rebuild, redeploy')
else:
    print(f'✅ All recent roles present ({len(dash.get("roles",[]))} total)')
"
```

---

## Seniority Rules (Critical for Dashboard Filtering)

The dashboard filters out "Analyst" and "Associate" tagged roles. Use buy-side seniority conventions:

| Title Contains | Seniority Tag | On Dashboard? |
|---|---|---|
| Portfolio Manager, Portfolio Strategist | Portfolio Manager | ✅ |
| Managing Director | Managing Director | ✅ |
| Director, Head of | Director | ✅ |
| Vice President, VP | Vice President | ✅ |
| Associate (at buy-side firm) | Vice President | ✅ |
| Analyst (at buy-side firm, non-junior) | Senior | ✅ |
| SVP, Senior Vice President | SVP/MD | ✅ |
| AVP | AVP | ✅ |
| Lead | Lead | ✅ |
| Manager | Manager | ✅ |
| Senior, Sr. | Senior | ✅ |
| Analyst (junior) | Analyst | ❌ |
| Associate (entry-level) | Associate | ❌ |

**Rule:** Never tag buy-side roles as "Analyst" or "Associate" — they get filtered from the dashboard. At buy-side firms, "Associate" = Vice President, "Analyst" = Senior.

---

## Vercel Deployment

The dashboard is deployed as a static site at **https://fi-jobs-dashboard.vercel.app/**.

### Deploying Updates
```bash
cd ~/dashboard/jpm-fi-jobs
# 1. Update data.json (via scraper refresh)
cd ~/pm_search_bot && source venv/bin/activate
python3 fi_job_scraper.py --refresh-md

# 2. Commit and push
cd ~/dashboard/jpm-fi-jobs
git add data.json
git commit -m "Refresh data — YYYY-MM-DD"
git push origin main

# 3. Deploy to Vercel
vercel --prod --yes --token "$(python3 -c "import json; print(json.load(open("/root/.openclaw/credentials/vercel.json"))[\"token\"])")"
```

Token stored at `/root/.openclaw/credentials/vercel.json`.

---

## Adding a New Firm

1. Find the career site URL and determine the ATS type (Workday, iCIMS, Eightfold, Oracle, etc.)
2. If Workday: find the tenant, wd version, and site name (see Workday API Reference above)
3. Test the API endpoint with curl
4. Filter for FI-relevant US roles
5. Get posted dates from detail API/pages
6. Add to `fi_jobs_raw.json`: `{"firm","title","location","seniority","link","tier","posted"}`
7. Add to `career_sites.json` with correct `type` and `url`
8. Run `python3 fi_job_scraper.py --refresh-md`
9. Deploy: `git add data.json && git commit -m "Add Firm" && git push && vercel --prod`

### Greenhouse Boards (proven 2026-06-07)

Greenhouse has a public boards API — no scraping needed.

**Step 1: Find the board slug**
```bash
# Company slug is in the URL: job-boards.greenhouse.io/{slug}
# Or: boards-api.greenhouse.io/v1/boards/{slug}/jobs
```

**Step 2: Fetch all jobs**
```bash
curl -s "https://boards-api.greenhouse.io/v1/boards/{slug}/jobs?content=true" | python3 -m json.tool
```
Each job has: `id`, `title`, `absolute_url`, `location.name`, `updated_at`, `content` (HTML job description).

**Step 3: Filter for FI/insurance PM roles**
```python
import requests

slug = "nationallifeinsurancecompany"  # example
r = requests.get(f"https://boards-api.greenhouse.io/v1/boards/{slug}/jobs?content=true", timeout=15)
jobs = r.json().get('jobs', [])

fi_kw = ['portfolio manager', 'fixed income', 'insurance', 'credit', 'investment',
         'asset management', 'risk', 'trading', 'ALM', 'capital markets', 'solutions',
         'annuity', 'pension', 'municipal', 'structured', 'mortgage', 'securities',
         'treasury', 'derivatives', 'hedging', 'RMBS', 'MBS', 'CLO', 'ABS']

us_states = ['NY','CA','TX','MA','CT','IL','FL','NJ','PA','WA','CO','GA','NC',
             'VA','DC','OH','MN','AZ','MI','OR','WI','IN','TN','MO','IA','Remote']

for j in jobs:
    title = j.get('title', '').lower()
    loc = j.get('location', {}).get('name', '')
    if any(kw in title for kw in fi_kw):
        if any(s in loc for s in us_states) or 'Remote' in loc:
            print(f"{j['title']:50s} | {loc:30s} | {j['absolute_url']}")
```

**Step 4: Get seniority from title**
Use the same seniority keyword matching as other scrapers (see Seniority Rules above). Greenhouse does not expose seniority in the API.

**Step 5: Add to `fi_jobs_raw.json`**
```python
import json

new_role = {
    "firm": "National Life Group",
    "title": "Sr. Director, Insurance Portfolio Management",
    "location": "New York, NY",  # clean to City, ST
    "seniority": "Director",
    "link": "https://job-boards.greenhouse.io/nationallifeinsurancecompany/jobs/4261859009",
    "tier": "top",
    "posted": "2026-06-01"  # from updated_at[:10]
}

with open('fi_jobs_raw.json') as f:
    raw = json.load(f)
existing = {(j['firm'], j['title']) for j in raw}
if (new_role['firm'], new_role['title']) not in existing:
    raw.append(new_role)
    with open('fi_jobs_raw.json', 'w') as f:
        json.dump(raw, f, indent=2)
```

**Step 6: Refresh and deploy**
```bash
cd ~/pm_search_bot && source venv/bin/activate
python3 fi_job_scraper.py --refresh-md
cd ~/dashboard/jpm-fi-jobs && git add -A && git commit -m "Add {firm}" && git push
vercel --prod --yes --token "$(python3 -c 'import json; print(json.load(open("/root/.openclaw/credentials/vercel.json"))["token"])')"
```

**Known Greenhouse slugs:**
| Firm | Slug |
|------|------|
| National Life Group | `nationallifeinsurancecompany` |
| Lincoln Financial | `lincoln` |
| AQR Capital | `aqr` |
| Point72 | `point72` |
| MGT Insurance | `mgtinsurance` |
| Jane Street | `janestreet` |
| KKR | `kkr` |
| Robinhood | `robinhood` |
| SoFi | `sofi` |
| Stripe | `stripe` |
| Coinbase | `coinbase` |
| Brex | `brex` |
| Mercury | `mercury` |
| Chime | `chime` |

**Bulk search across all known slugs:**
```python
import requests

slugs = ['nationallifeinsurancecompany','lincoln','aqr','point72','mgtinsurance',
         'janestreet','kkr','robinhood','sofi','stripe','coinbase','brex','mercury','chime']

fi_kw = ['portfolio manager','fixed income','insurance','credit','investment',
         'asset management','trading','ALM','capital markets','annuity','derivatives']

for slug in slugs:
    try:
        r = requests.get(f"https://boards-api.greenhouse.io/v1/boards/{slug}/jobs", timeout=5)
        if r.status_code == 200:
            for j in r.json().get('jobs', []):
                if any(kw in j['title'].lower() for kw in fi_kw):
                    loc = j.get('location',{}).get('name','')
                    print(f"{slug:25s} | {j['title']:50s} | {loc}")
    except: pass
```

---

## MetLife (Avature ATS) Notes

MetLife uses Avature, a client-side rendered ATS that extensively uses JavaScript. Direct API calls are not publicly exposed, requiring Playwright for reliable interaction.

### How to scrape MetLife (Avature):
1. Use Playwright to navigate to the careers page (`https://www.metlifecareers.com/en_US/ml/SearchJobs`).
2. Interact with the dynamic filter elements using their specific Avature IDs:
   - **Country**: Use the `select2` dropdown for filter ID `12310[]`, selecting `15` for "United States".
   - **Career Level**: Use the `<select>` element for filter ID `8862`, setting values for "Mid-Level" (`446411`) and "Executive" (`446413`).
   - **Area of Expertise**: Use the `<select>` element for filter ID `7518`, setting values for "Finance" (`81480`) and "Investment Management" (`7418`).
   - **Work Arrangement**: Interact with the `select2` dropdown for filter ID `4877[]`, selecting all available options.
   - **Keywords**: Enter relevant FI keywords (from `career_sites.json` `fi_keywords`) into the main search input (name `search`).
3. The search action is a form POST, and pagination works via `jobRecordsPerPage` and `jobOffset` parameters in subsequent GET requests.
4. Extract job details (title, URL, location, posted date, description) from the rendered HTML.

### Key details:
- The scraper should handle filtering and pagination dynamics through Playwright browser automation.
- Ensure `fi_keywords`, `exclude_keywords`, `us_locations`, and `exclude_seniority` filters from `career_sites.json` are applied to the extracted job data.
- Seniority assignment should follow the rules in the "Seniority Rules" section.

```bash
cd ~/pm_search_bot && source venv/bin/activate
python3 fi_job_scraper.py --firm "MetLife"
```

---

## Known Issues & Bug Fixes

### J.P. Morgan: Missing/Broken Links & Pagination (2026-07-27)

**Symptom:** Initially, many J.P. Morgan roles on the dashboard had missing or non-functional apply links. Further investigation revealed that the scraper was either not capturing links for certain roles or that the links were aggressively pruned due to `requests.head()` failing.

**Root Cause:**
1.  **Oracle HCM Virtualized Rendering:** J.P. Morgan's Oracle ATS uses a virtualized DOM, meaning job links only appear for items visible in the current viewport. The previous scraper collected links from the final viewport state, losing links for roles that scrolled out of view.
2.  **Incorrect Link Pruning:** The link validation step, which used `requests.head()`, incorrectly marked many valid API-generated links as stale or broken because many ATS platforms block HEAD requests.
3.  **API Pagination Misunderstanding:** The Oracle HCM REST API's pagination mechanism for `recruitingCEJobRequisitions` was not correctly implemented, leading to only the first 25 requisitions being fetched (due to `data.hasMore` always being `false` when using a `limit` of 25, or `count: 1` when using a `limit` of 200 via `finder` parameter).

**Fixes Implemented:**
1.  **Oracle HCM REST API Integration:** The `scrape_oracle_ats` function was entirely rewritten to bypass DOM scraping for links. It now uses Playwright to load the initial page (to establish session cookies), then makes authenticated `fetch()` calls directly to the Oracle HCM REST API (`/hcmRestApi/resources/latest/recruitingCEJobRequisitions`).
2.  **Robust Pagination:** The API calls now correctly paginate through all available requisitions by incrementing the `offset` parameter until no new roles are returned (`batchCount === 0`), effectively fetching all relevant jobs (e.g., 298 requisitions found for "Fixed Income").
3.  **Direct Link Construction:** Apply links are now reliably constructed using the `Id` field from the API response: `https://jpmc.fa.oraclecloud.com/hcmUI/CandidateExperience/en/sites/{site_number}/job/{Id}`.
4.  **Date Format Conversion:** Posted dates from the API (YYYY-MM-DD) are converted to MM/DD/YYYY to match dashboard expectations.
5.  **Seniority Harmonization:** Ensured the `assign_seniority` function correctly processes titles to assign appropriate seniority (e.g., "Associate" to "Vice President" for buy-side roles) immediately after scraping and before dashboard filtering.

**Result:**
-   **31 J.P. Morgan roles** were identified as FI-relevant and are now correctly displayed on the dashboard.
-   **All 31 roles** now have working apply links and accurate posted dates.
-   **10 new J.P. Morgan roles** from the past 2 weeks were successfully identified and included.
-   The dashboard at `https://fi-jobs-dashboard.vercel.app` reflects these updates.

### New roles silently dropped by dashboard filter (2026-07-20)

**Symptom:** Recently posted roles exist in `fi_jobs_raw.json` but don't appear on the dashboard.

**Root cause:** Roles scraped from Workday APIs return with `seniority: "—"`. The `generate_dashboard_data()` function estimates `minComp` from seniority — when seniority is `—`, it defaults to $150k, which is below the $200k `MIN_COMP` filter. The roles get silently dropped with no error.

**Fix:** Assign seniority from job titles immediately after merging new data (Step 6b in REFRESH_PROMPT.txt). The seniority assignment function maps title keywords to buy-side seniority levels:
- "SVP" → SVP
- "Director" → Director
- "Vice President" / "VP" → Vice President
- "Associate" → Vice President (buy-side convention)
- "Analyst" → Senior (buy-side convention)
- "Manager" → Manager
- "Principal" → Principal

**Prevention:** Always run the dashboard verification step (Step 13 in REFRESH_PROMPT.txt) after deploy. It compares recent roles in `fi_jobs_raw.json` against `data.json` and flags any that are missing.

**Affected roles (2026-07-20):** 14 roles across Blackstone, Voya, Capital Group, PIMCO, Invesco, Apollo — all fixed and deployed.
