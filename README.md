# OFFERZZZ

An n8n workflow that watches company job boards, scores every new posting against a candidate profile using Claude, and writes the results to Airtable — so a job search runs on a schedule instead of on willpower.

**The problem it solves:** checking twenty career pages by hand, every day, is exactly the kind of task people abandon in week two. This does it at 100% consistency and adds a reasoning layer on top.

## How it works

```
Schedule Trigger (daily)
  ├─ Fetch Greenhouse Jobs ─┐
  ├─ Fetch Lever Jobs ──────┼─► Merge ─► Pre-Filter ─► Deduplicate
  ├─ Fetch Ashby Jobs ──────┘              │              │
  ├─ Airtable: Search Jobs ────────────────┘              │
  └─ Airtable: Lookup Companies                           ▼
                                              Loop Over Items (batched)
                                                    │
                                    Build Prompt ─► Call Claude ─► Parse Scores
                                                                       │
                                                    Drop Fails ─► Enrich ─► Airtable
```

1. **Fan-out fetch** — hits the public job APIs of three ATS platforms (Greenhouse, Lever, Ashby) in parallel. Each returns a different shape; each node normalizes to a common schema.
2. **Pre-filter** — a deterministic pass that drops roles by title keyword and location whitelist. This runs *before* the LLM, so obvious non-matches never cost a token.
3. **Deduplicate** — checks each posting's `external_id` against what's already in Airtable, so re-runs are idempotent and only genuinely new roles proceed.
4. **Batched scoring** — surviving jobs are chunked and sent to `claude-haiku-4-5` in batches, with a system prompt that pins the output to exactly four lines per job. Batching is what keeps this economical: one call scores many roles.
5. **Parse and enrich** — responses are parsed back into structured fields (fit score, recommended action, role family, one-line rationale). Malformed rows get dropped rather than corrupting the table.
6. **Write** — new records land in Airtable, ready to triage.

## Design notes

A few decisions that mattered more than they look:

- **Cheap filter before expensive filter.** The keyword pre-filter is unglamorous but removes the large majority of postings for free. The LLM only ever sees plausible candidates.
- **Batching over per-item calls.** Scoring jobs one API call at a time is the obvious implementation and roughly an order of magnitude more expensive. The `Build Prompt` node assembles many jobs into a single numbered request.
- **A rigid output contract.** The system prompt demands exactly four prefixed lines per job. Free-form output would have made `Parse Scores` a parsing nightmare.
- **Fail open, not dirty.** `Drop Fails` discards anything that didn't parse cleanly instead of writing partial records into Airtable.

## Setup

You need an [n8n](https://n8n.io) instance, an Airtable base, and an Anthropic API key.

1. **Import the workflow** — in n8n, *Workflows → Import from File → `workflow.json`*.

2. **Create the Airtable base** with two tables:
   - `Jobs` — the output table (title, company, location, url, fit score, action, role family, rationale, date found)
   - `Companies` — your watchlist

3. **Add credentials** in n8n (*Settings → Credentials*): an Airtable Personal Access Token and an Anthropic API credential. Then open each Airtable node and the `Call Claude` node and select them — the placeholders in `workflow.json` (`REPLACE_WITH_YOUR_CREDENTIAL_ID`) do not resolve on their own.

4. **Point at your base** — in each Airtable node, replace `REPLACE_WITH_YOUR_BASE_ID` / `REPLACE_WITH_YOUR_TABLE_ID` by picking your base and table from the dropdown.

5. **Set your target companies** — edit the `companies` array at the top of each fetch node. The `token` is the board slug from the company's careers URL:

   ```js
   const companies = [
     { name: 'Example Co', token: 'exampleco' },  // boards.greenhouse.io/exampleco
   ];
   ```

6. **Set your candidate profile** — in `Build Prompt`, replace the `Candidate: [YOUR PROFILE ...]` placeholder with a description of yourself and the roles you're targeting. This is what the scoring is relative to.

7. **Tune the filters** — `Pre-Filter` holds an `excludeTitles` array and an `allowedLocations` whitelist. Both are worth adjusting to your search.

## A note on what's in this repo

This is a redacted export. Credential IDs, the Airtable base and table IDs, the original target-company list, and the candidate profile have all been replaced with placeholders. No API keys were ever stored in the workflow file itself — n8n keeps credentials server-side and the export only ever referenced them by ID.

## License

MIT
