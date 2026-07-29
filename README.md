# warmstart

**From cold job board to first-round conversation.**

An n8n workflow that watches company job boards on a schedule, scores every new posting against a candidate profile using Claude, and writes the results to Airtable with a recommended next action. The goal is to cut the distance between a role existing and you being in a conversation about it.

Checking twenty career pages by hand every day is the kind of task people abandon in week two. This runs at full consistency and adds a reasoning layer on top.

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

1. **Fan-out fetch.** Three nodes hit the public job APIs of Greenhouse, Lever, and Ashby in parallel. Each platform returns a different shape. Each node normalizes to a common schema.
2. **Pre-filter.** A deterministic pass drops roles by title keyword and location whitelist. It runs before the model, so obvious non-matches never cost a token.
3. **Deduplicate.** Each posting's `external_id` is checked against what Airtable already holds. Re-runs are idempotent. Only genuinely new roles continue.
4. **Batched scoring.** Surviving jobs are chunked and sent to `claude-haiku-4-5`. The system prompt pins output to exactly four lines per job.
5. **Parse and enrich.** Responses are parsed back into structured fields: fit score, recommended action, role family, and a one-line rationale. Malformed rows are dropped rather than written.
6. **Write.** New records land in Airtable with a recommended action attached, ready to act on.

## Design notes

Decisions that mattered more than they look:

- **Cheap filter before expensive filter.** The keyword pre-filter is unglamorous. It removes most postings for free, so the model only ever sees plausible candidates.
- **Batching over per-item calls.** Scoring one job per API call is the obvious implementation. It costs roughly an order of magnitude more. The `Build Prompt` node assembles many jobs into one numbered request.
- **A rigid output contract.** The system prompt demands exactly four prefixed lines per job. Free-form output would make `Parse Scores` unreliable.
- **Drop bad rows instead of writing them.** `Drop Fails` discards anything that did not parse cleanly. Partial records in Airtable are worse than missing ones.
- **Recommend an action, not just a score.** Every row gets Apply Now, Network First, Watchlist, or Skip. A score alone still leaves you deciding what to do next.

## Setup

You need an [n8n](https://n8n.io) instance, an Airtable base, and an Anthropic API key.

1. **Import the workflow.** In n8n, go to Workflows, then Import from File, and select `workflow.json`.

2. **Create the Airtable base** with two tables:
   - `Jobs` for output: title, company, location, url, fit score, action, role family, rationale, date found.
   - `Companies` for your watchlist.

3. **Add credentials** in n8n under Settings, then Credentials. You need an Airtable Personal Access Token and an Anthropic API credential. Open each Airtable node and the `Call Claude` node and select them. The placeholders in `workflow.json` (`REPLACE_WITH_YOUR_CREDENTIAL_ID`) do not resolve on their own.

4. **Point at your base.** In each Airtable node, replace `REPLACE_WITH_YOUR_BASE_ID` and `REPLACE_WITH_YOUR_TABLE_ID` by picking your base and table from the dropdown.

5. **Set your target companies.** Edit the `companies` array at the top of each fetch node. The `token` is the board slug from the company's careers URL.

   ```js
   const companies = [
     { name: 'Example Co', token: 'exampleco' },  // boards.greenhouse.io/exampleco
   ];
   ```

6. **Set your candidate profile.** In `Build Prompt`, replace the `Candidate: [YOUR PROFILE ...]` placeholder with a description of yourself and the roles you want. Scoring is relative to this.

7. **Tune the filters.** `Pre-Filter` holds an `excludeTitles` array and an `allowedLocations` whitelist. Both are worth adjusting to your search.

## What is in this repo

This is a redacted export. Credential IDs, the Airtable base and table IDs, the original target-company list, and the candidate profile are all placeholders.

No API keys were ever stored in the workflow file. n8n keeps credentials server-side and the export references them by ID only.

## License

MIT
