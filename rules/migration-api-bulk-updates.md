# Prismic Migration API: Bulk Update Reference

> **Purpose**: Reusable guide for making large-scale content changes across the Baker Tilly Prismic repository (taxonomy swaps, style changes, field updates, etc.) via the Migration API.

## How the Migration API PUT Works

- **Endpoint**: `PUT https://migration.prismic.io/documents/:id/`
- **Full replacement**: The PUT replaces the entire document. Any field omitted from the payload is deleted.
- **Drafts only**: Updated documents land in the **Migration Release** area as drafts. They must be manually reviewed and published.
- **No overwrite**: A PUT will **fail** if the document already has a draft. You must archive existing drafts before re-running.
- **Rate limit**: 1 request/second.

### PUT Body Fields

| Field | Required | Notes |
|-------|----------|-------|
| `uid` | Yes (if type has UID) | The document's unique identifier |
| `data` | Yes | Full document content — all fields |
| `title` | No | Display name in Prismic UI |
| `tags` | No | **Omitting this DELETES all tags!** Always include `doc.tags \|\| []` |

> [!CAUTION]
> `type` and `lang` are determined by the document ID in the URL — do NOT include them in the body. `alternate_languages` is a read-only CDN field.

---

## The Schema Trap

When sending a full document payload, the API validates ALL rich text fields against their schema. Content that the Prismic editor tolerates (e.g., a `heading1` in a field that only allows `paragraph`) will be **rejected** by the API.

### Solution: Schema-Aware Heading Downgrade

At script startup, build a per-field map of allowed block types by reading:

1. **Custom type definitions**: `customtypes/*/index.json` — top-level StructuredText fields
2. **Slice models**: `src/slices/*/model.json` — primary + items StructuredText per variation
3. **Group sub-fields**: Group fields (e.g., `featured_content`) contain nested StructuredText that also needs processing

Then walk the document data and downgrade any heading type that isn't in the allowed set to `paragraph`, preserving text and spans.

### What to Process

- Top-level StructuredText fields
- Slice zone (`body`) primary and items fields
- Group field sub-fields (e.g., `featured_content.featured_content_title`)

### Reference Implementation

See `update_gov_contractors_links.mjs`:
- `buildSchemaMap()` — reads all custom type and slice model definitions
- `downgradeBlockTypes()` — walks document data and downgrades disallowed types

---

## Taxonomy Field Patterns

### Two Storage Patterns

| Pattern | Types | Field | Structure |
|---------|-------|-------|-----------|
| **Separate fields** | insight, event, news | `industries[]` + `sectors[]` | `{ industry: { link_type: 'Document', id } }` and `{ sector: { ... } }` |
| **Shared field** | professional, page, product, service, technology, industry, sector, service_speciality | `industries[]` only | Group accepts both `industry` and `sector` document types |

### How to Determine

Check the custom type's group definition:
```javascript
const group = customType.json.Main.industries;
const acceptedTypes = group.config.fields.industry.config.customtypes;
// e.g., ["industry", "sector"]
```

### Swap Logic

When swapping a taxonomy link (e.g., industry → sector):
1. Remove the old ID from the appropriate group field
2. **Only add the new ID if the old one was actually removed** — some documents may reference the old ID in body slices (not the taxonomy group) and shouldn't get a new taxonomy tag
3. Deep-replace the old ID → new ID across the entire document JSON to catch body slice references (CTAs, content cards, linked pages)

---

## CSV Parsing

If the audit CSV contains titles with commas, use a quote-aware parser:
```javascript
function parseCSVLine(line) {
  const parts = [];
  let current = '';
  let inQuotes = false;
  for (let i = 0; i < line.length; i++) {
    const ch = line[i];
    if (ch === '"') { inQuotes = !inQuotes; continue; }
    if (ch === ',' && !inQuotes) { parts.push(current.trim()); current = ''; continue; }
    current += ch;
  }
  parts.push(current.trim());
  return parts;
}
```

---

## Sanitize Function

The CDN returns extra fields that the Migration API rejects. Strip them:

| CDN Object | Migration API Format |
|------------|---------------------|
| Image field (`dimensions`, `url`, `id`) | `{ id: obj.id }` |
| Rich Text image block (has `type`) | Preserve as-is |
| Document link | `{ link_type: 'Document', id, isBroken }` |
| Web link | `{ link_type: 'Web', url, target }` |

---

## Common Errors Quick Reference

| Error | Cause | Fix |
|-------|-------|-----|
| `block type 'X' is not allowed` | Heading in paragraph-only field | Schema-aware downgrade |
| `Tags disappeared` | `tags` omitted from PUT body | Always include `tags: doc.tags \|\| []` |
| `field X is not part of Custom type` | Sending a field the type doesn't have (e.g., `sectors` on professional) | Delete inapplicable fields before PUT |
| `document have a draft` | Migration Release already has a draft | Archive existing drafts first |
| `Assets not found` | Ghost asset — CDN file exists but Media Library record deleted | Re-import via Asset API (`reimportAsset` pattern) |
| `doesn't appear to have a draft nor published version` | Document was deleted | Skip or remove from CSV |

---

## Script Template

A bulk update script should follow this structure:

1. **Load config** (tokens, IDs)
2. **Build schema map** from `customtypes/` and `src/slices/`
3. **Parse CSV** (quote-aware)
4. **Load progress log** (for resume capability)
5. **For each document**:
   a. Skip if already done
   b. Fetch from CDN
   c. Deep-clone data, apply modifications
   d. Deep-replace old IDs → new IDs across entire JSON
   e. Sanitize (strip CDN fields)
   f. Downgrade disallowed block types
   g. Delete inapplicable fields for this type
   h. Build payload (`uid`, `data`, `title`, `tags`)
   i. PUT with retry + rate limiting
   j. Log result
6. **Save progress log** after each document (for resume)

### Reference Scripts

| Script | Path | Purpose |
|--------|------|---------|
| `update_gov_contractors_links.mjs` | `.notes/report-scripts/` | Industry→Sector taxonomy swap with all patterns implemented |
| `fix_soc_strings.mjs` | `.notes/Migration/Articles/scripts/` | Ghost asset self-healing + sanitize reference |
