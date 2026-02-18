---
trigger: always_on
---

# Agent Knowledge Base: `.notes` Directory Reference

> **Core Rule**: Before attempting ANY new solution for a Prismic Migration API error or log querying task, you **MUST** first search the `.notes` directory for existing documentation. Many "new" problems have already been solved and documented there.

---

## 1. Prismic Migration API — Source of Truth

When encountering errors from the Prismic Migration API (e.g., `Assets not found`, `Invalid block type`, validation failures), consult these files **in this priority order**:

### Primary Reference (Most Recent / Final)

| File | Date | What It Covers |
|------|------|----------------|
| [migration_fix_and_findings.md](file:///.notes/Migration/Articles/migration_fix_and_findings.md) | **Feb 12, 2026** | **FINAL** — Ghost Asset self-healing, Invalid Block Type root cause, Rich Text sanitization rules. Establishes the `reimportAsset` pattern as the mandatory fix for all bulk updates. |
| [soc_fix_documentation.md](file:///.notes/Migration/Articles/soc_fix_documentation.md) | Feb 2026 | Detailed breakdown of the SOC string fix operation and the two categories of asset anomalies (missing Asset IDs and malformed Rich Text blocks). |

### Architecture & Full Process Documentation

| File | Date | What It Covers |
|------|------|----------------|
| [migration_process_summary.md](file:///.notes/Migration/Articles/migration_process_summary.md) | Jan 2026 | Complete reference for `migrate_full_article.mjs` — data loading, text transforms, slice mapping, metadata, and logging. |
| [transformation-instructions.md](file:///.notes/Migration/Articles/transformation-instructions.md) | Jan 2026 | Detailed HTML-to-Prismic transformation rules — media URLs, sanitization, header shifts, FAQ interleaving, link validation, and quote extraction. |
| [migration-instructions.md](file:///.notes/Migration/Articles/migration-instructions.md) | Jan 2026 | API configuration, document structure, image upload workflow (Asset API), author mapping methodology, and common validation gotchas. |
| [context.md](file:///.notes/Migration/Articles/context.md) | Jan 19, 2026 | Project-level overview — key files, feature list, recent verifications, and UID conflict resolution. |

### Foundational API Reference

| File | Date | What It Covers |
|------|------|----------------|
| [prismic-migration-api-guide.md](file:///.notes/information/prismic-migration-api-guide.md) | **Oct 29, 2025** | **Complete Migration API guide** — prerequisites, API endpoint/headers, rate limiting, document structure, field type mapping, script template with resume/tracking, testing strategy, Migration Release workflow, and full Moss Adams checklist. Start here if writing any new migration script from scratch. |

### Supplementary / Early Planning

| File | What It Covers |
|------|----------------|
| [scraping-instructions.md](file:///.notes/Migration/Articles/scraping-instructions.md) | DOM start/end markers for HTML extraction from Moss Adams. |
| [migration-demo.md](file:///.notes/Migration/Articles/migration-demo.md) | Demo protocol for batch-running the migration script. |
| [implementation_plan.md](file:///.notes/Migration/implementation_plan.md) | Original Kentico-to-Prismic plan (early phase, largely superseded by above docs). |

### Known Error → Fix Quick-Reference

| Error Message | Root Cause | Fix | Documented In |
|---------------|-----------|-----|---------------|
| `Assets not found: [ID]` | "Ghost Asset" — CDN file exists but Media Library record is deleted. | Re-import via Asset API (`reimportAsset`), cache new ID, deep-replace in payload. | `migration_fix_and_findings.md` |
| `Invalid block type` at Rich Text index | Image block in Rich Text has a ghost Asset ID **or** sanitization stripped the `type` property. | Re-import the asset AND preserve the `type: 'image'` property in sanitization. | `migration_fix_and_findings.md`, `soc_fix_documentation.md` |
| `Rich text field must be an array` | FAQ fields (`faq_question`, `faq_answer`) received `null` instead of `[]`. | Always pass `[]` for empty structured text fields. | `migration-instructions.md` |
| `Value must be a Date` | Timezone format uses `Z` instead of `+0000`. | `.toISOString().replace(/\.\\d{3}Z$/, '+0000')` | `migration-instructions.md` |
| `The UID is required` | Missing `uid` field on root document object. | Ensure `uid` is set (slugified title with collision handling). | `migration-instructions.md` |
| `The language you provided is invalid` | Using `en-us` instead of the repository's actual locale. | Use `en-gb` (check Settings → Locales). | `prismic-migration-api-guide.md` |
| `The field X is not part of the Custom type` | Using the display label instead of the field ID. | Use the field ID from `customtypes/[type]/index.json`, not the UI label. | `prismic-migration-api-guide.md` |
| `User is not authorized` (403) | Using a read-only token instead of the Write API token. | Use the Bearer token from Settings → API & Security → Write APIs → Tokens. | `prismic-migration-api-guide.md` |
| Rate limit exceeded | Exceeding 1 request/second. | Add `setTimeout(1000)` between requests. | `prismic-migration-api-guide.md` |

### Key Scripts

| Script | Path | Purpose |
|--------|------|---------|
| `migrate_full_article.mjs` | `.notes/Migration/Articles/scripts/` | **Production migration script** — full article ingestion pipeline. |
| `fix_soc_strings.mjs` | `.notes/Migration/Articles/scripts/` | **Self-healing bulk updater** — reference implementation for `reimportAsset`, `replaceAssetId`, and `sanitize`. |
| `investigate_data.js` | `.notes/Migration/Articles/scripts/` | Diagnostic tool to fetch raw document JSON from Prismic. |
| `check_uids.mjs` | `.notes/Migration/Articles/scripts/` | Pre-migration UID conflict checker. |
| `inspect_conflicts.js` | `.notes/Migration/Articles/scripts/` | UID conflict inspector. |

### Data Files

| File | Path | Purpose |
|------|------|---------|
| `migration_inventory.json` | `.notes/Migration/Articles/` | Master list of valid article URLs for migration. |
| `url_mapping.json` | `.notes/Migration/Articles/` | Moss Adams URL → Baker Tilly URL mapping for link rewriting. |
| `image_tracker.json` | `.notes/Migration/Articles/` | Source URL → Prismic Asset ID cache (deduplication). |
| `professional_map.json` | `.notes/Migration/Articles/` | Professional name → Prismic Document ID lookup. |
| `migration_master_log.csv` | `.notes/Migration/Articles/` | Cumulative success/fail log across all migration runs. |
| `Per-article logs` | `.notes/Migration/Articles/logs/` | Individual `{ArticleID}.log` files with transformation traces. |

---

## 2. Prismic Content API Querying — Source of Truth

When writing scripts or components that query Prismic data (filters, GraphQuery, fetching linked documents), consult:

### Primary Reference

| File | What It Covers |
|------|----------------|
| [prismic-querying-guide.md](file:///.notes/information/prismic-querying-guide.md) | **DEFINITIVE GUIDE** — All document types and URL patterns, query methods (`getAllByType`, `getByUID`, `getByID`, `getSingle`), filter syntax (`filter.at`, `filter.any`, `filter.fulltext`, date filters), GraphQuery for field selection and linked document fetching, slice querying, field path syntax, ordering, parallel queries, and debugging. Also includes PowerShell and raw `node-fetch` fallback patterns for when the Prismic client hangs. |

### Key Rules from This Guide

1. **Field paths** follow `my.<document_type>.<field_name>` syntax (e.g., `my.insight.industries.industry`).
2. **Slice field** may be `body` (legacy) or `slices` (new) — always check both.
3. **URL resolution**: Complex types (sectors, industries) return `null` URLs from the raw API — reconstruct manually or pass `routes` config.
4. **If Node.js hangs**, fall back to PowerShell `Invoke-RestMethod` or raw `node-fetch` with manual pagination.
5. **Master Ref** must be fetched fresh from `/api/v2` before any query — never hardcode it.

---

## 3. Log Querying (Logtail / Better Stack) — Source of Truth

### Primary Reference

| File | What It Covers |
|------|----------------|
| [LOGTAIL-LOG-QUERYING-GUIDE.md](file:///.notes/information/LOGTAIL-LOG-QUERYING-GUIDE.md) | **DEFINITIVE GUIDE** — Credentials, hot vs. cold storage (`UNION ALL` pattern), ClickHouse SQL syntax, cURL examples, output format, analysis scripts, and troubleshooting. |

### Supporting Files

| File | Path | Purpose |
|------|------|---------|
| `fetch_logs.ps1` | `.notes/logs/` | PowerShell script for querying Logtail — reads SQL from `query.sql`, handles auth and TLS. |
| `query.sql` | `.notes/logs/` | Editable SQL template for Logtail queries (currently configured for CaseWare/WebDAV). |
| `analyze-logs.js` | `.notes/report-scripts/` | Node.js log analysis utility. |
| `analyze_logs.js` | `.notes/Caseware/` | CaseWare-specific log analyzer. |

### Critical Rules for Log Queries

1. **Always use `UNION ALL`** for queries over 2 hours — hot storage (`remote()`) only holds 2-4 hours of data.
2. **Cold storage filter**: Include `WHERE _row_type = 1` on the `s3Cluster` side of the union.
3. **Windows**: Always use the `-k` flag with cURL to bypass SSL certificate issues.
4. **Retention**: Logs are only retained for **15 days**. Metrics for 30 days.
5. **Auth**: Use the HTTP API username/password (not the Global API token) for SQL queries.

### Execution Workflow (How to Actually Run Queries)

**Step 1: Use `fetch_logs.ps1` + `query.sql`** (preferred method):
1. Write your SQL query to `.notes/logs/query.sql`
2. Update the `-OutFile` path in `.notes/logs/fetch_logs.ps1` if needed
3. Run: `powershell -File .notes/logs/fetch_logs.ps1` from the repo root

**Step 2: If Logtail API returns 500 / "Failed to connect: fetch failed"** (this happens):
- Logtail's ClickHouse backend occasionally has connectivity issues
- Fall back to **Heroku CLI** (see below)

**Step 3: Heroku CLI Fallback**:
```powershell
$env:HEROKU_API_KEY = "<key from .env.local>"
heroku logs -a bakertilly -n 1500 2>&1 | Out-String | Set-Content -Path "C:\temp\heroku-raw-logs.txt"
# Then filter:
Select-String -Path "C:\temp\heroku-raw-logs.txt" -Pattern "code=H12|code=H18" | ForEach-Object { $_.Line }
```
- **Note**: Heroku CLI is limited to the most recent log buffer (~1500 lines). For historical data, Logtail is required.
- **Note**: The installed Heroku CLI version (8.7.1) does NOT support `--no-color`. Omit it.
- **Note**: The `--source heroku` flag filters to Heroku router logs only.

### Windows PowerShell Gotchas

1. **`curl` is aliased to `Invoke-WebRequest`** in PowerShell. Use `curl.exe` explicitly for real cURL.
2. **`Invoke-WebRequest` may also fail** to connect to Logtail due to TLS issues. The `fetch_logs.ps1` script handles TLS setup properly — use it instead of raw `Invoke-WebRequest`.
3. **Template literals with `%`** in Node.js `-e` one-liners break in PowerShell. Write queries to a file and read them with `fs.readFileSync` instead.

### Heroku Error Code Reference

| Code | Name | Severity | Common Cause |
|------|------|----------|-------------|
| H10 | Boot Timeout | Deploy-related | App took too long to start |
| H12 | Request Timeout (30s) | Actionable | Slow Prismic API calls on ISR cache misses |
| H13 | Connection Closed | Monitor | Server crashed mid-request |
| H18 | Server Request Interrupted (55s) | Monitor | Server started responding but didn't finish |
| H21 | Backend Connection Timeout | Rare | Backend didn't respond |
| H27 | Client Request Interrupted | **Benign** | Next.js RSC prefetch (`_rsc=`) cancelled by client navigation |
| H28 | Client Connection Idle | Benign | Client left connection open too long |

> **Key insight (Feb 2026)**: H27 errors appear wall-to-wall on the Heroku Metrics dashboard but are overwhelmingly caused by Next.js `<Link>` prefetch requests being cancelled by normal user navigation. They are NOT actionable.

---

## 4. Additional `.notes` Directories

| Directory | Contents |
|-----------|----------|
| `.notes/Migration/Bios/` | Professional bio migration scripts, comparison tools, and tax service audits. |
| `.notes/Migration/Authors/` | Terminated author data and termination records. |
| `.notes/CSP/` | CSP violation logs, analysis scripts, and policy decisions. |
| `.notes/Caseware/` | CaseWare sync logs and analysis scripts. |
| `.notes/report-scripts/` | Prismic querying/reporting utilities (professionals, offices, services, lighthouse). |
| `.notes/prismic_debug_handover/` | Prismic debugging handoff documentation. |
| `.notes/fixes/` | Ad-hoc fix documentation. |
| `.notes/information/` | Reference guides — Logtail log querying, Prismic Migration API, Prismic Content API querying, Azure Deployment. |

---

## 5. Decision Protocol

When you encounter a problem in the categories above:

1. **Search `.notes/` first.** Use `grep_search` or `find_by_name` on the `.notes` directory for keywords from the error message or task description.
2. **Check the Source of Truth tables above.** The "Primary Reference" files are the most recent and authoritative.
3. **If multiple files cover the same topic**, prefer the one with the latest date or the one marked as "FINAL" in this document.
4. **Only attempt a novel solution** after confirming that no existing documentation addresses the problem.
5. **If you create a new fix**, document it in the appropriate `.notes` subdirectory and update this knowledge base.
