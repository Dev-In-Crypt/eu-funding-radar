# EU Funding Radar — n8n workflow (MVP)

Daily monitor for EU grants and cascade-funding calls. Pulls from two
sources, normalizes them into one schema, deduplicates, uses an LLM to
pull structured eligibility criteria out of free-text call descriptions
(with a citation check so nothing is invented), filters against a
recipient profile, and posts a digest to Slack.

This is a scoped-down build of a larger architecture (12 sources, 3
separate workflows, embedding-based dedup, a metrics dashboard — see
`ARCHITECTURE.md`). This MVP implements the checkpoint from that
document's own build order: two sources, one working end-to-end path,
before scaling out. Everything past that checkpoint is listed under
**Roadmap** below, not silently skipped.

## What it does

1. **Collect** — fetches open calls from the EU Funding & Tenders
   Portal (SEDIA search API) and the Cascade Funding portal.
2. **Normalize** — maps both sources' very different shapes onto one
   canonical call object.
3. **Deduplicate** — two levels: same call seen again on a later run
   (hash of source + external id), and the same call published on more
   than one source (normalized title + deadline + budget bucket).
4. **Extract** — an LLM reads the free-text call description and pulls
   out structured eligibility criteria (consortium size, TRL range,
   co-financing %, etc). Every extracted field must come with a
   verbatim quote from the source text; a field is dropped if its
   quote isn't an exact substring of the original — the model can't
   silently assert something the text doesn't say.
5. **Filter** — matches the extracted criteria against a recipient
   profile (hard-excludes vs. soft flags shown to the reader).
6. **Notify** — posts up to 5 matching calls to Slack, sorted by
   deadline. An empty result is a valid, expected outcome and still
   sends one line — silence is never ambiguous with "the workflow
   broke".

## Setup

1. **Create two Data Tables** in your n8n instance (Data → Data
   Tables), named exactly `raw_calls` and `calls`, with these columns:

   **raw_calls**
   | column | type |
   |---|---|
   | id | string |
   | source_code | string |
   | external_id | string |
   | cross_key | string |
   | title | string |
   | programme | string |
   | type | string |
   | url | string |
   | summary | string |
   | amount_min_eur | number |
   | amount_max_eur | number |
   | deadline | string |
   | deadline_type | string |
   | topics | string (JSON-encoded array) |
   | raw_text | string |
   | fetched_at | string |

   **calls** — same columns as `raw_calls`, plus:
   | column | type |
   |---|---|
   | eligibility | string (JSON-encoded object) |
   | dropped_fields | string (JSON-encoded array) |
   | extraction_confidence | number |
   | completeness | number |
   | first_seen_at | string |
   | last_seen_at | string |
   | notified | boolean |

2. **Credentials** — add an Anthropic credential (used by the
   "Anthropic Chat Model" node) and a Slack credential (used by "Post
   Digest"). Point `channelId` at the channel you want the digest in.

3. **Check the Cascade Funding selectors** — the "Cascade Funding:
   Parse Listing" node uses placeholder CSS selectors
   (`.call-item h3`, `.call-item .deadline`, `.call-item .summary`).
   Open the live listing page, inspect the real markup, and update
   them before relying on this source.

4. **Import** `eu-funding-radar.json` and activate.

## Design notes

- **Citation-checked extraction.** The Information Extractor node asks
  for a `*_quote` field alongside every extracted value. "Validate
  Citations & Score" checks each quote is a literal substring of the
  call's raw text; anything that fails is dropped rather than kept
  with an asterisk. This mirrors the same no-unverified-claims rule
  used elsewhere in this portfolio (see the ICP-matching project).
- **`alwaysOutputData` on "Get Unnotified Calls".** Without it, zero
  pending rows means zero items means the downstream nodes never run
  at all — and the "no matches today" message never gets sent. Turning
  it on makes "nothing to report" an explicit, visible output instead
  of silence.
- **Profile as config.** The matching rules live in one `PROFILE`
  object inside "Apply Recipient Profile". `profiles/*.json` shows the
  intended config shape for more than one recipient — the workflow
  itself only reads the inlined one for now (see Roadmap).
- **Fail loud on HTML.** "Cascade Funding: Normalize" throws if the
  page returns zero matched elements, instead of quietly reporting
  "no calls today" — a selector break and an actually-quiet week
  should never look the same.

## Limitations

- The SEDIA search endpoint used by "FTOP: Search Calls" is not
  officially documented by the European Commission; the query shape
  is reverse-engineered from public usage and can change without
  notice. Unlike the HTML source, this node does not currently detect
  and fail loudly on a schema change — that's asymmetric and worth
  fixing before this runs unattended for long.
- Level-2 dedup is simplified: the first-seen source wins and later
  duplicates are dropped, rather than keeping the higher-completeness
  record and tracking aliases.
- No level-3 (embedding-similarity) dedup for reworded titles.
- Only 2 of the 12 sources in the full architecture are implemented.
- No circuit breaker, schema-drift alerting, or metrics dashboard yet.
- `sme-team.json` is an example of the config format, not currently
  wired up — switching profiles today means editing the `PROFILE`
  constant in code.

## Roadmap

Straight from the source architecture doc, in the order it proposes:
split into three workflows (Collector / Normalizer / Notifier) for
per-source failure isolation, add the remaining 10 sources (TED,
CORDIS, OpenAIRE, EIC, EIT, CINEA, Eureka, EEA Grants, ESA, Interreg),
level-3 dedup, call versioning with an `updated` event for deadline
and budget changes, and a metrics dashboard per source.

See `ARCHITECTURE.md` for the full design this MVP is a checkpoint
toward.
