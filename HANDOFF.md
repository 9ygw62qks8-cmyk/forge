# Forge — Handoff Document

**Purpose:** Drop this into a new Claude conversation so the assistant has full
context to continue working on Forge with you. Paste the relevant parts (or the
whole file) at the start of the new chat.

---

## WHO YOU ARE

Investor/operator, solo founder, on Mac (macOS). Non-technical but capable —
got Node.js installed, ran scrapers, dealt with macOS Gatekeeper permissions.
Building **Forge** as a 90-day MVP. Don't want a co-founder. Don't want to
show this to anyone yet ("when more polished"). Building first, then will
talk to potential users.

## WHAT FORGE IS

A federal tech transfer marketplace — a portal to find/license technology from
NASA, DOE national labs, and (eventually) universities, NIH, NIST, FFRDCs.

**The product surfaces:**
- **Search** with keyword + filters (Institution, Category, Center, Type)
- **AI Brief** — auto-generated plain-English summary on each tech detail page
- **AI Match** — describe a problem, get top 5 ranked tech matches with reasoning
- **Discovery Scan** — describe your company, get a comprehensive ranked list (30–50)
  with signal-typed reasoning (strong fit / adjacent / stretch / long shot)
- **Business Case Builder** — autogenerate startup thesis OR custom company synergy
  analysis for any tech
- **Persistent Profile** — save company description once, auto-fills Discovery Scan
  and Business Case Builder
- **Inquiry flow** — structured request routing to TTO emails per institution

**Strategic positioning** (decisions already made):
- Sequencing: NASA first → DOE labs (Battelle is leverage point — manages PNNL,
  ORNL, INL, NREL) → mid-tier R1 universities (Florida, Purdue, Georgia Tech,
  Wisconsin, Minnesota) → NIH/NIST → FFRDCs (SRI, Battelle, MITRE)
- Skip top performers (Stanford, MIT) — they have brand, won't prioritize
- Skip bottom — structural problems
- Sweet spot: frustrated middle-tier institutions
- Monetization (acknowledged, not built): token-tiered free vs paid.
  - Free: unlimited search, 5 AI Match/mo, 2 Discovery Scans/mo, 1 saved profile
  - Pro $X/mo: unlimited everything, watchlist alerts, priority TTO routing
  - Enterprise: team accounts, USPTO enrichment, license comparables (the moat)

---

## CURRENT BUILD STATE

### Files (all in `/mnt/user-data/outputs/`)

| File | Purpose | Status |
| --- | --- | --- |
| `forge.jsx` | The artifact. 2.5MB single-file React component. | ✅ Working — drops into Claude artifacts or React app |
| `nasa-scraper.js` | NASA T2 API scraper (~1,673 records) | ✅ Run successfully — data is wired into artifact |
| `ornl-scraper.js` | ORNL HTML scraper (~2,100 records) | 🔄 Currently running on user's machine (~50 min total) |
| `doe-patents-scraper.js` | DOE patents via OSTI API (~14,000 records, all 17 labs) | ⚠️ Just written, NOT YET TESTED. First version had wrong endpoint, this is the fix |
| `inl-scraper.js` | INL-specific HTML scraper | ❌ ABANDONED. Tested, broken: INL's portal doesn't paginate or expose detail pages publicly. INL coverage comes via DOE Patents API instead. |
| `README.md` | Architecture, schema, run instructions | ✅ Up to date |

### Catalog State (in artifact)

- **NASA: 1,673 records** (real, scraped from public T2 API). Categories
  normalized 100% (was 57% in "other" — fixed via case-insensitive map).
  613 patents + 1,060 software releases across 11 NASA centers.
- **DOE: 19 curated demo records** across 9 labs (ORNL, ANL, NREL, INL, SAND,
  LLNL, LBNL, PNNL, BNL). These are placeholder demo data, NOT real scraped.
  Will be replaced when ORNL + DOE Patents scrapers complete.
- **Total: 1,692 records**

### Scrapers Status (running in user's terminals)

1. **ORNL scraper:** running, ~50 min total. Output → `./data/ornl.json`.
   Should produce ~2,100 records.
2. **DOE Patents scraper:** ready to run after fixing the endpoint. Output →
   `./data/doe-patents.json`. Should produce ~14,000 records in ~2 min.
3. **INL scraper:** ready to run, but selectors are unverified. May 403 or
   return zero URLs — has clear error messages if so.

---

## ARCHITECTURE — CANONICAL SCHEMA v1.0 [LOAD-BEARING]

The most important architectural decision. Every record from every source —
NASA, DOE, MIT, anyone — gets normalized to this shape. Source-specific
quirks live in adapters; the rest of the system never sees them.

```
{
  forgeId: 'forge_<institution>_<unit>_<id>',
  source: {
    institution: 'NASA' | 'DOE' | 'NIH' | 'University' | ...,
    subUnit: 'Oak Ridge National Laboratory',     // human-readable
    subUnitCode: 'ORNL',                          // controlled vocab
    nativeId: 'GSC-TOPS-1' or 'ORNL-202301',     // source's own ID
    sourceUrl: 'https://...',
    syncedAt: ISO date,
    confidence: 'high' | 'medium' | 'low',
  },
  type: 'patent' | 'software',
  title, abstract, brief: null,                  // brief is AI-generated lazy
  category: 'aerospace',                          // canonical (controlled vocab)
  categoryRaw: 'Aeronautics',                     // preserved from source
  tags: [...],
  trl: { value: 1-9, source: 'stated' } | null,
  availability: [{                                // multi-row crucial
    fieldOfUse, exclusivity, geographic, status, notes,
  }],
  patent: { number, inventors, filingDate, grantDate, ... } | null,
  software: { releaseType, repositoryUrl, license } | null,
  exportControl: { itar: bool, ear: bool, notes },
  contacts: [{ role: 'concierge'|'tto-officer', email, ... }],
  imageUrl: string | null,
  schemaVersion: '1.0',
}
```

**Forge canonical category vocab:** aerospace, propulsion, materials_coatings,
sensors_instruments, electronics, manufacturing, robotics_automation,
software_data, communications, biomedical_health, energy_power,
environment_climate, mechanical_systems, imaging_optics, operations,
design_integration, other.

The artifact builds `TECHNOLOGIES` by combining `TECHNOLOGIES_RAW` (NASA) +
`DOE_CANONICAL` (DOE) and adding flat-field aliases (`id`, `center`,
`description`, `category` as label string) for backward compat with existing
UI components. In production: drop flat fields, refactor UI to canonical only.

---

## KEY UI/UX DECISIONS

- **Editorial/research-archive aesthetic**, not techy. Fonts: Fraunces (serif
  display), IBM Plex Sans (body), IBM Plex Mono. Palette: cream (#F4EFE6)
  background, deep ink (#161513), terracotta accent (#BC4A23), moss
  secondary (#5C6A3C).
- **AI Brief** uses Anthropic API directly via fetch with model
  `claude-sonnet-4-20250514`. No API key (handled by Claude artifact runtime).
  Sentinel `<json>` tags + per-field char limits prevent truncation.
- **AI Match** does keyword pre-filter to top 150 candidates, then sends to
  Claude for semantic ranking with reasoning. Token-budget safe at scale.
- **Discovery Scan** does the same two-stage retrieval but returns 30–50
  results with signal categorization (strong_fit / adjacent / stretch /
  long_shot) and per-result one-line reasoning.
- **Persistent profile** uses `window.storage` (Anthropic artifact storage
  API). Saves to key `forge:profile`. Auto-fills Business Case custom mode
  with green "Auto-filled" indicator.
- **SearchView pagination**: 50 records per page, "Load 50 more" button.
  Resets when query/filters change.
- **Institution filter** chip at the top of search filters (NASA / DOE).
- **Institution badge** on every TechCard — black NASA, dark navy DOE.

---

## TECHNICAL NOTES (DON'T LOSE THESE)

- **`callClaude` helper** returns `{ ok, text }` or `{ ok, error }` — always
  surfaces errors in UI.
- **`extractJson` helper** uses real brace-matching with string state tracking
  (not naive lastIndexOf) — robust to Claude prose preamble.
- **All AI prompts** use sentinel `<json>...</json>` tags.
- **NASA scraper** hits `https://technology.nasa.gov/api/api/{patent|software}/{keyword}`
  with ~70 keyword seeds, dedupes by forgeId, 1s rate limit. All 11 NASA
  centers + FAA + HDQS edge cases handled. TTO emails embedded per center.
  Agency concierge: `Agency-Patent-Licensing@mail.nasa.gov`.
- **ORNL scraper** does HTML scraping in two stages: paginate listing
  (?page=0..210), then fetch each `/technology/{ID}` detail page. Resume-capable
  via `./data/ornl.progress.json`. TTO contact: `partnerships@ornl.gov`,
  865-574-1051.
- **DOE Patents scraper** uses `https://www.osti.gov/doepatents/api/v1/records`
  (the documented public OSTI API). Pages start at 1 (not 0 — that was the bug
  in the first version). 200 records per page. Reads `X-Total-Count` header.
  Auto-detects which lab from `research_orgs` string.
- **INL scraper** uses `https://inl.gov/technology-deployment/?_paged=N` for
  listings. Selectors UNVERIFIED — site may have changed structure. Run
  `--max=5 --verbose` first to test.
- **macOS quarantine** on downloaded files: user has resolved this already —
  Files & Folders permission for Terminal in System Settings, plus
  `xattr -d com.apple.quarantine` for downloaded files.

---

## DOE LABS / API LANDSCAPE (FOR REFERENCE)

DOE has multiple federation portals, each with its own access pattern:

- **OSTI APIs** (public, no auth): `osti.gov/api/v1/records` (general),
  `osti.gov/doepatents/api/v1/records` (patents). What we use.
- **Lab Partnering Service (LPS)** at `labpartnering.org`: SPA, content via JS,
  needs API key from `developer.nlr.gov/signup`. NREL is rebranding to NLR;
  `developer.nrel.gov` shuts down May 29, 2026. URL patterns:
  `/technology-summaries/{UUID}`, `/patents/{PATENT_NUMBER}`, `/labs/{CODE}`.
- **VIPS (Visual Intellectual Property Search)** at `vips.pnnl.gov`: SPA,
  14,000+ patents + 6,200+ software, all 17 DOE labs + sites. Source data
  feeds from USPTO + DOE CODE/OSTI — same source we hit directly.
- **DOE CODE API** at `osti.gov/doecodeapi/services/...`: documented but
  endpoint paths unclear. The first scraper attempt (`doe-code-scraper.js`)
  used wrong path and got 404'd. NOT in current scraper bundle.
- **Argonne** at `anl.gov/partnerships/express-licensing-technologies`: 403's
  automated fetches (bot detection). Would need browser-grade scraping.
- **INL** at `inl.gov/technology-deployment`: scraper written but unverified.

DOE lab TTO emails (in scraper code):
ORNL `partnerships@ornl.gov`, ANL `partners@anl.gov`,
NREL `technology.transfer@nrel.gov`, INL `partnerships@inl.gov`,
SAND `ip@sandia.gov`, LLNL `partnerships@llnl.gov`,
LBNL `ipo@lbl.gov`, PNNL `techtransfer@pnnl.gov`,
BNL `techtransfer@bnl.gov`, LANL `techtransfer@lanl.gov`,
SLAC `techtransfer@slac.stanford.edu`, FNAL `tt@fnal.gov`,
TJNAF `ipo@jlab.org`, PPPL `techtransfer@pppl.gov`,
SRNL `srnl.partnerships@srnl.doe.gov`, AMES `techtransfer@ameslab.gov`,
NETL `techtransfer@netl.doe.gov`.

---

## KEY CONTACTS / IDENTIFIERS

- **NASA agency concierge:** Agency-Patent-Licensing@mail.nasa.gov
- **NASA T2 director:** Daniel Lockney (target for outreach)
- **DOE FLC Federation:** https://flcbusiness.federallabs.org/
- **LPS:** https://www.labpartnering.org/
- **VIPS:** https://vips.pnnl.gov/home
- **OSTI DOE CODE:** https://www.osti.gov/doecode/

---

## WHAT'S NEXT

When you start the new chat, here's the priority queue:

### Immediate (when scrapers finish)

1. **Send `data/ornl.json` to assistant** — wire it into the artifact
   replacing the 19 curated DOE samples. Should be ~2,100 ORNL records.
2. **Run `doe-patents-scraper.js`** with `--max=20 --verbose` smoke test.
   If endpoint works (it should, was researched and verified), run full
   scrape. Output: `data/doe-patents.json`.
3. **Send `data/doe-patents.json` to assistant** — wire it in alongside
   ORNL. Combined catalog: NASA 1,673 + ORNL ~2,100 + DOE Patents ~14,000
   = **~17,800 federal technologies**. INL coverage comes through this
   automatically via the OSTI API's `research_orgs` field — INL records
   auto-detect to `subUnitCode: 'INL'`.

### Strategic (build is real enough to test now)

5. **Draft Lockney email** — outreach to NASA T2 director Daniel Lockney.
   Establish credibility with NASA before showing the broader DOE catalog.
6. **Buyer interview script** — 5 prospective corporate scouts / startup
   founders to test the value prop on. Don't show the product yet; do
   problem-validation interviews first.
7. **Token tiering implementation** — gate AI Match/Discovery Scan behind
   counters that read/write to `window.storage`. Foundation already in
   place via persistent profile.
8. **DOE software** — separate scraper using `osti.gov/api/v1/records`
   with `product_type=Software` filter. Adds ~6,200 records. Lower
   priority than patents (most are open source).

### Down-funnel (after first user signal)

9. **University adapter** — start with mid-tier R1 (Florida, Purdue, Georgia
   Tech, Wisconsin, Minnesota). Each has its own portal; same per-source
   adapter pattern as ORNL.
10. **USPTO enrichment** — pull in claim text, citations, prosecution
    history. This is the data moat for Enterprise tier.
11. **License comparables database** — hand-curate 50–100 known DOE/NASA
    licensing deals and the financial terms (where public). The "what
    should I expect to pay" question is the hardest for buyers.

---

## OPEN QUESTIONS

- Will the DOE Patents scraper actually return ~14,000 records, or has
  OSTI rate-limited / restructured? Need to run smoke test to confirm.
- Will INL's site let the scraper through, or 403? If 403, skip it for
  now — DOE Patents API will cover most of INL's patent corpus anyway.
- When NASA T2 / DOE TTOs see this, will they cooperate or feel like
  Forge is competing with their official portals? Need to position as
  "we drive licensing inquiries to your office, not away from it."

---

## IF YOU'RE CLAUDE READING THIS

Pick up where the user left off. The user is building agentically. They
care about: making the catalog real, not pretty; shipping deliverables they
can run/test; honest assessment of what works and what doesn't. They don't
want hand-wringing about edge cases — they want the next thing built.

The user's tone: direct, casual, pragmatic. No emojis. They appreciate
honesty about technical risks ("untested," "may need adjustment"). They
appreciate strategic framing ("here's why this is worth doing").

When in doubt: ship the simplest thing that works, flag what's untested,
move on. The artifact is in the working directory. The scrapers run on
the user's machine. Output files come back to you for wiring in.
