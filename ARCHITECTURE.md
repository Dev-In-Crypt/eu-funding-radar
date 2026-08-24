# n8n Pipeline: EU Funding Radar

Portfolio project. A generic monitor for European grants, cascade-funding
calls, and tenders. Not tied to any single country, legal entity, or
person: the recipient's profile is a config, not a hardcoded assumption.

This is the full target architecture. The MVP shipped in this repo
(`eu-funding-radar.json`) implements the checkpoint described in section
10, step 5, with 2 of the 12 sources and the LLM extraction step pulled
forward. Everything past that checkpoint here is roadmap, not implemented
— see the main README for exactly what's built vs. planned.

## 1. What this demonstrates

The portfolio value isn't that the workflow calls some APIs — anyone can
do that. The value is in the three places where this kind of pipeline
usually breaks:

1. **Normalizing inconsistent schemas.** Six sources, six incompatible
   formats, one canonical object on the way out.
2. **Stateless deduplication.** Every run is isolated; history lives in
   external storage; the same call appearing on three aggregators
   collapses into one record.
3. **LLM-based eligibility extraction with validation.** The model pulls
   structured eligibility conditions out of free text; every field is
   checked against a schema, and anything that fails validation is
   flagged, not silently dropped or silently kept.

Plus an adapter architecture: adding a source is a new config row, not a
new branch in the workflow.

## 2. Architecture

Three separate workflows instead of one monolith. This matters both for
reliability and as a demonstration:

```
WF-1  Collector      schedule, one run per source, writes to raw
WF-2  Normalizer     triggered by new raw rows, normalizes, dedupes,
                      runs LLM extraction, writes to calls
WF-3  Notifier       schedule, reads calls, applies the recipient
                      profile, sends the digest
```

The split buys: one source failing doesn't take down the others,
re-running normalization doesn't require re-fetching, and changing the
recipient profile doesn't touch data collection.

## 3. Sources

Each source is a row in a `sources` table with: code, transport type,
endpoint, schedule, parser, rate limits.

### Primary

| Code | What it provides | Transport | Frequency |
|---|---|---|---|
| `ftop` | The official EU portal: Horizon, Digital Europe, EDF, LIFE calls, including cascade funding | JSON search API (SEDIA) | daily |
| `ted` | Tenders and procurement across all EU countries | Search API v3, POST with expert query | daily |
| `cordis` | Funded projects, consortia, amounts | Bulk CSV/XML dumps | weekly |
| `openaire` | Projects and grants across many funders, including non-EU | Documented REST API | weekly |
| `cascadefunding` | Showcase of FSTP calls from Horizon projects | HTML | daily |
| `eic` | EIC Accelerator, Transition, Pathfinder, cut-off dates | HTML plus RSS | weekly |

### Secondary — usually missing from comparable aggregators

This is what separates a portfolio project from a tutorial.

| Code | What it provides | Why it's useful |
|---|---|---|
| `eureka` | Eurostars and GlobalStars | Countries outside the EU, including associated ones |
| `cinea` | Innovation Fund, CEF, LIFE | Large amounts, a separate portal from FTOP |
| `eit` | KIC calls: Digital, Manufacturing, Urban Mobility, Health | Distributed through their own sites, don't appear in FTOP |
| `eeagrants` | EEA and Norway Grants | Entirely outside the EU-portal ecosystem |
| `esa` | ESA OSIP, ARTES, BIC | Space and downstream applications, its own portal |
| `interreg` | Cross-border programmes | Regional, often lower competition |

12 sources total. Adding a 13th requires no workflow changes.

### Transport notes

- **FTOP**: the JSON search endpoint is public but undocumented and can
  change. A fallback and drift alert are mandatory. In a portfolio this
  is a plus — it shows the work of handling an unstable API.
- **TED**: anonymous, no key required. Limits: 700 requests/minute, 600
  downloads per IP per 6 minutes, 3 parallel downloads. Needs a throttle
  node.
- **CORDIS**: not an API — archives. Download weekly, unzip, load in
  full.
- **HTML sources**: selectors break. Every parser must verify it found
  more than zero elements and fail explicitly, rather than returning an
  empty list silently.

## 4. Canonical schema

Everything is normalized to one object. Fields missing from a given
source stay `null` and are flagged in `completeness`.

```json
{
  "id": "sha256(source_code + external_id)",
  "source_code": "ftop",
  "external_id": "HORIZON-CL4-2026-DIGITAL-01-15",
  "title": "...",
  "programme": "Horizon Europe",
  "type": "grant | cascade | tender | prize | equity",
  "url": "https://...",
  "summary": "...",
  "amount_min_eur": 50000,
  "amount_max_eur": 150000,
  "amount_total_eur": 2000000,
  "opens_at": "2026-07-01",
  "deadline": "2026-10-15",
  "deadline_type": "single | multiple | continuous",
  "eligibility": {
    "consortium_required": true,
    "min_partners": 3,
    "min_countries": 3,
    "countries": ["EU27", "UA", "NO"],
    "applicant_types": ["sme", "startup", "research", "public", "ngo"],
    "trl_min": 5,
    "trl_max": 8,
    "cofinancing_pct": 30
  },
  "topics": ["ai", "cybersecurity"],
  "cpv": ["72000000"],
  "language": "en",
  "raw_hash": "...",
  "first_seen_at": "...",
  "last_seen_at": "...",
  "completeness": 0.82,
  "extraction_confidence": 0.91
}
```

`completeness` is the share of meaningful fields that are filled in.
Records below 0.4 go to a separate manual-review queue instead of the
digest.

## 5. Deduplication

Three levels, increasing in cost.

1. **By source ID.** `sha256(source_code + external_id)`. Catches the
   same call reappearing on the next run.
2. **By cross-source key.** Normalized title plus deadline plus amount.
   Catches the same FSTP call showing up in both `cascadefunding` and
   `ftop`. The record with higher `completeness` wins; the rest are
   written to `aliases`.
3. **By title/description embedding.** Cosine similarity above 0.92 plus
   a matching deadline. Catches reworded titles. Expensive, so it only
   runs on records that passed the first two levels and fall in the same
   time window.

Storage: Postgres or Supabase. n8n Data Tables work for a demo but will
hit limits at 12 sources.

One more detail worth showing: calls **change**. Deadlines shift, budgets
change. So a record isn't simply skipped as a duplicate — `raw_hash` is
compared, and on a mismatch a new version is written to `call_versions`
and an `updated` event fires. A deadline-extension notification is often
more valuable than a new-call notification.

## 6. LLM extraction

One of the few tasks where an LLM is justified: eligibility criteria are
written in prose, differently in every programme, and regex can't parse
them.

Rules:

- The model receives the call text and a JSON Schema, and returns only
  the structure — no prose.
- Every extracted field comes with a quote from the source text.
- A validator checks that the quote is actually a substring of the
  source. If it isn't: the field is nulled and `extraction_confidence`
  drops.
- Fields that can't be backed by a quote stay `null`. No guessing.
- The call only runs on new records, and only after deduplication —
  otherwise you pay for the same text three times.

Same principle as in the ICP engine: no claims without a checkable
source. In a portfolio, show the validator, not the prompt.

## 7. Recipient profile

Filtering lives in config so the pipeline isn't tied to one applicant.
Demo profiles in the repo: solo developer, SME with a team, research
institute, municipality.

```yaml
profile: solo-software
countries: [ES, UA, PL]
entity_types: [sme, startup, natural_person]
consortium: false
max_partners: 2
trl: [3, 6]
cofinancing_max_pct: 20
amount: [20000, 300000]
deadline_min_days: 10
domains_include: [ai, data, cybersecurity, digital, open-source]
domains_exclude: [agriculture, construction, clinical-trials, maritime-hardware]
requires_physical_infra: false
```

The matcher returns not a boolean but an explanation: which conditions
passed and which failed. A call that fails one soft condition is shown
with a flag, not dropped.

## 8. Output

- Digest to Slack, Telegram, or email, capped at 5 items.
- A row in Notion or Google Sheets as a working database.
- A webhook for external consumers.
- A separate channel for `updated` events: deadline shifts, budget
  changes.
- An empty digest is a valid result, sent as one line.

## 9. Reliability

What separates a portfolio workflow from a demo:

- **Retry with exponential backoff** on every HTTP node, tuned to each
  source's own limits.
- **Circuit breaker**: a source that fails three runs in a row gets
  disabled and raises an alert, instead of making noise every hour.
- **Schema drift detection**: if a source's field set changes, the
  workflow doesn't stay silent or write garbage — it fails with a clear
  message.
- **Idempotency**: re-running the same period doesn't create duplicates
  or resend notifications.
- **Dry-run mode**: everything executes, nothing is sent, results land
  in a table for review.
- **Metrics**: per source — record count, response time, invalid-record
  rate. A separate dashboard, and a good portfolio screenshot.

## 10. Build order

1. DB schema plus a `sources` table with two entries: `ftop` and
   `cascadefunding`.
2. WF-1 Collector for these two, writing to `raw`.
3. WF-2 Normalizer: canonicalization plus dedup levels 1 and 2, no LLM
   yet.
4. WF-3 Notifier with one demo profile.
5. **Checkpoint: the end-to-end run works, duplicates don't get through.**
   Only now start expanding.
6. LLM extraction with the citation validator.
7. TED and CORDIS: different transport, bulk downloads and throttling.
8. The remaining 8 sources, one at a time.
9. Dedup level 3, versioning, the `updated` event.
10. Metrics and dashboard.

## 11. What to show in the portfolio README

- A diagram of the three workflows and the data flow.
- Before/after normalization: three raw responses from different
  sources and one canonical object.
- A dedup example: one call found by three sources, collapsed into one
  record.
- An example of an LLM extraction rejected because its citation check
  failed.
- A screenshot of the metrics dashboard.
- An honest section on limitations: the undocumented FTOP endpoint, the
  fragility of HTML parsers, TED's rate limits.

That last point isn't a weakness. An engineer who names their system's
limitations reads better than one who claims everything always works.
