# Forge

**Federal technology transfer marketplace.** Discovery + business case generation + execution for licensing tech from NASA, DOE labs, NIH, NIST, and major research universities.

This bundle is the MVP scaffold. Four files:

| File | Purpose |
| --- | --- |
| `forge.jsx` | The artifact. Working demo with embedded NASA + DOE data, AI features, business case builder, discovery scan. Drop into Claude artifacts or a React app to render. |
| `nasa-scraper.js` | Standalone Node.js scraper for NASA. Pulls the full T2 catalog (~1,673 records) via NASA's public API. Writes canonical JSON to `./data/nasa.json`. |
| `ornl-scraper.js` | Standalone Node.js scraper for ORNL. Walks ORNL's tech transfer site (~2,100 records) by HTML scraping. Two-stage: listings then details. Resume-capable. |
| `README.md` | This file. Architecture, schema, how to run, how to add new sources. |

---

## How to run the scrapers

### NASA

```bash
node nasa-scraper.js                  # scrape everything (~10-15 min, ~1,673 records)
node nasa-scraper.js --verbose        # with retry logging
node nasa-scraper.js --only=patents   # patents only
node nasa-scraper.js --only=software  # software only
```

NASA exposes a public JSON API at `technology.nasa.gov/api`. The scraper queries 70+ keyword seeds and dedupes. No API key, no auth, no rate limit issues at 1 req/sec.

### ORNL (Oak Ridge National Laboratory)

```bash
node ornl-scraper.js                  # full scrape (~55 min, ~2,100 records)
node ornl-scraper.js --listings-only  # IDs + titles only (~4 min, 2,100 stubs)
node ornl-scraper.js --max=50         # cap to N records (testing)
node ornl-scraper.js --resume         # continue from saved progress
node ornl-scraper.js --verbose        # detailed logs
```

ORNL has no JSON API. The scraper does two stages: (1) walk the paginated listing pages to collect all `/technology/{id}` URLs, (2) fetch each detail page and parse out title, description, benefits, applications, market category, contact email, and researchers from the Drupal-rendered HTML. Saves progress every 50 records — Ctrl+C and `--resume` is safe.

The two scrapers output to `./data/nasa.json` and `./data/ornl.json` respectively. Both files use the same canonical schema, so you can concatenate them and drop into the artifact's `TECHNOLOGIES` array.

To use scraped data in the artifact, replace the embedded `TECHNOLOGIES_RAW` array in `forge.jsx` with the JSON output. In production, the artifact would hit your own API instead of embedding the array.

---

## The canonical schema (Forge v1.0)

The most important architectural decision in this codebase. Every record from every source — NASA, DOE, MIT, Stanford, anyone — gets normalized to this shape. Source-specific quirks live in adapters; the rest of the system never sees them.

```js
{
  forgeId: "forge_nasa_msfc_tops_93",     // your namespace, never changes
  source: {
    institution: "NASA",                   // top-level institution
    subUnit: "Marshall Space Flight Center",
    subUnitCode: "MSFC",
    nativeId: "MFS-TOPS-93",               // the source's own ID
    sourceUrl: "https://technology.nasa.gov/patent/MFS-TOPS-93",
    syncedAt: "2026-05-06T12:34:00Z",
    confidence: "high"                     // data quality marker
  },
  type: "patent",                          // patent | software | know-how
  title: "...",
  abstract: "...",                         // original from source
  brief: null,                             // AI-generated, lazy
  category: "propulsion",                  // Forge controlled vocab
  categoryRaw: "Propulsion",               // original string preserved
  tags: [],
  trl: { value: 6, source: "stated" },     // or null
  availability: [                          // multiple rows per record!
    {
      fieldOfUse: "all",                   // or 'medical', 'defense', etc.
      exclusivity: "either",               // exclusive | non-exclusive | either
      geographic: "global",
      status: "available",
      notes: "..."
    }
  ],
  patent: {                                // null for software
    number: null,
    inventors: [],
    filingDate: null,
    grantDate: null,
    expirationDate: null,
    family: []
  },
  software: {                              // null for patents
    releaseType: "Open Source",
    repositoryUrl: "...",
    license: null
  },
  exportControl: {
    itar: false,
    ear: false,
    notes: "..."
  },
  contacts: [
    {
      role: "concierge",                   // concierge | tto-officer
      name: null,
      email: "Agency-Patent-Licensing@mail.nasa.gov",
      phone: null,
      organizationUnit: "NASA Technology Transfer Program"
    }
  ],
  imageUrl: null,
  schemaVersion: "1.0"
}
```

### Why each field matters

**`forgeId` vs `source.nativeId`** — IDs across institutions collide. NASA "TOP-1" and a university "TOP-1" need to be distinguishable. The Forge ID is your namespace; the native ID is preserved separately for display and TTO communication.

**`source.confidence`** — NASA data is structured and reliable. Some university TTO exports will be CSVs from someone's spreadsheet. Track which records are high-quality so the UI can flag low-confidence data and prioritize cleanup.

**`category` (controlled vocab) vs `categoryRaw`** — Search across institutions only works if categories are normalized. "Aerospace" at NASA, "Aerospace Engineering" at MIT, "Aerodynamics" at Purdue — same Forge category. Keep the raw string so you never lose information when you re-derive the taxonomy.

**`availability` is an array, not a single field** — A single patent can be exclusively available in medical, non-exclusively in industrial, and unavailable in defense, all at once. Get this right or the schema is broken. NASA's API doesn't expose this granularity yet, but DOE and university TTOs will.

**`exportControl` as first-class** — DOE, NIH, NASA defense-adjacent tech is often ITAR/EAR controlled. Foreign entities legally cannot license it. Has to be a filterable, prominent field.

**`schemaVersion`** — When you change the canonical schema in 6 months, you need a migration path. Versioning every record is the cheap insurance that makes that possible.

---

## How to add a new source

The architecture is built around adapters. NASA is the first. To add DOE, NIH, MIT:

1. **Write the adapter.** Like `nasaToCanonical()` in the scraper. Function signature: `(rawRecord) => CanonicalRecord | null`. Map their fields to the canonical schema. Extend `CATEGORY_MAP` with their category strings.

2. **Write the scraper.** Copy `nasa-scraper.js`, replace the API calls and the adapter. Output goes to `./data/{source}.json`.

3. **Drop the JSON in.** The artifact loads from arrays of canonical records. The UI doesn't know or care which source produced any record — it just renders source.subUnit and source.institution as badges.

The whole UI, the AI features, the business case generator — none of it changes when you add a new source. That's the value of the canonical schema.

### Source priority (build order)

1. ✅ **NASA** — single agency, public API, government IP (no inventor splits), standardized templates, T2 office actively wants more commercialization. Easiest start.
2. **DOE national labs** — 17 labs, massive unlicensed pool. **Battelle is the leverage point** (manages PNNL, ORNL, INL, NREL — one relationship unlocks four labs).
3. **Mid-tier R1 universities** — Florida, Purdue, Georgia Tech, Wisconsin, Minnesota. Real research output, starved for qualified inbound, will actually call back. Avoid Stanford/MIT/Columbia early — they have brand and pipeline, won't prioritize you.
4. **NIH / NCI** — biotech-heavy, lots of CRADAs, distinct workflow.
5. **NIST, USDA, FFRDCs (SRI, Battelle, MITRE), spinout-friendly research institutes**.
6. **Top universities** — last, on your terms, after you have deal flow they want access to.

---

## Architecture (current)

```
forge.jsx           [Artifact: UI + embedded sample data]
  ├─ Schema docs (JSDoc types in code comments)
  ├─ NASA adapter (showing how raw → canonical)
  ├─ Sample canonical records (~30, hand-picked)
  └─ React UI components (search, detail, AI Match, business case)

nasa-scraper.js     [Standalone Node script]
  ├─ Same canonical schema definition
  ├─ Same NASA adapter
  ├─ HTTP layer with retries + polite rate limiting
  └─ Outputs ./data/nasa.json
```

## Architecture (production direction)

```
Backend
  ├─ /scrapers/{nasa,doe,nih,...}/    # one per source, run on cron
  ├─ /adapters/{nasa,doe,nih,...}/    # source → canonical
  ├─ /enrichment/                     # USPTO patent data, license comparables
  ├─ /db/postgres                     # canonical records
  ├─ /search/typesense (or algolia)   # fast keyword + faceted
  └─ /api                             # serves the frontend

Frontend (Next.js or similar)
  └─ Same component tree as forge.jsx, just hitting /api instead of an array

Workers
  ├─ Brief generator (Claude API, async, persisted)
  ├─ AI Match scorer (cached per problem statement hash)
  └─ Business case generator (per (tech × company) pair)

Integrations
  ├─ NASA TOPS sheet links (already in artifact)
  ├─ USPTO patent enrichment (claims, family, citations, expiration)
  ├─ SEC R&D parsing (for company autofill in synergy analysis)
  └─ FOIA-able federal license records (for comparable terms — moat)
```

## What's done in the MVP

- NASA adapter + canonical schema
- Working scraper for full NASA catalog
- Search + filter UI (by category, NASA center, type)
- AI plain-English brief generation per technology
- AI Match (reverse search: describe a problem, find tech)
- Business Case Builder (auto-thesis + custom synergy analysis)
- TTO contact routing (concierge + per-center fallback)
- Persistent saves across sessions

## What's intentionally not done

- No NDA / e-sign flow
- No live API hits from the artifact (CORS + sandboxed iframe)
- No USPTO enrichment
- No license comparables database (the long-term moat)
- No buyer accounts / CRM / deal tracking
- No second source plugged in yet (DOE adapter is the next deliverable)

---

## Notes for whoever builds the production version

The schema is the load-bearing decision. Everything else is replaceable. If you change the canonical schema after building three adapters, you re-do all three. So: extend it before changing it. Add new fields freely; remove fields only with a migration plan.

The brief and business case prompts in `forge.jsx` are tuned for sentinel-tag JSON output (`<json>...</json>`) rather than tool-use, because the artifact runtime calls `/v1/messages` directly. In production with the SDK, switch to tool-use with a JSON schema — it eliminates the entire class of "Claude's output didn't parse" bugs.

The AI Match feeds the entire technology list to Claude every time. That works for ~30 records. At 2,300+, switch to: (a) embed every tech, (b) embed the user's problem, (c) cosine similarity in vector space, top-N to Claude for ranking with reasoning. Same UX, scales.

---

*v0.1 · MVP scaffold · NASA only · 2026*
