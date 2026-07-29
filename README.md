# warmstart

**From cold job board to first-round conversation.**

An n8n workflow that watches company job boards on a schedule, scores every new posting against a candidate profile using Claude, and writes the results to Airtable with a recommended next action. The goal is to cut the distance between a role existing and you being in a conversation about it.

Checking twenty career pages by hand every day is the kind of task people abandon in week two. This runs at full consistency and adds a reasoning layer on top.

## What you need before you start

This is a workflow file, not an app. You cannot clone this repo and run it. It has to be imported into n8n, and it writes to your Airtable base using your own API keys.

| Requirement | Why | Cost |
|---|---|---|
| An [n8n](https://n8n.io) instance | Runs the workflow. Use n8n Cloud or self-host the free community edition with Docker. | Free self-hosted, or paid cloud |
| An [Airtable](https://airtable.com) account | Stores the scored jobs. The free tier is enough. | Free |
| An [Anthropic API key](https://console.anthropic.com) | Scores the postings. Billed per use. | A few cents per run |

You do not need accounts with Greenhouse, Lever, or Ashby. Those job board APIs are public and need no key.

Budget 30 to 45 minutes for first-time setup, most of it building the Airtable tables.

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
2. **Pre-filter.** A deterministic pass drops roles by title keyword and location whitelist. It runs before the model, so obvious non-matches never cost a token. On a typical board this removes about 70% of postings.
3. **Deduplicate.** Each posting's `External ID` is checked against what Airtable already holds. Re-runs are idempotent. Only genuinely new roles continue.
4. **Batched scoring.** Surviving jobs are chunked and sent to `claude-haiku-4-5`. The system prompt pins output to exactly four lines per job.
5. **Parse and enrich.** Responses are parsed back into structured fields: fit score, recommended action, role family, and a one-line rationale. Rows that fail to parse are marked `Rejected` and dropped rather than written.
6. **Write.** New records land in Airtable with a recommended action attached.

## Setup

### Step 1: Build the Airtable base

Do this first. The workflow writes to specific column names, and a mismatch means every write fails silently.

Create a base with two tables.

**Table `Companies`** (your watchlist):

| Column | Type |
|---|---|
| Company Name | Single line text |

**Table `Jobs`** (the output). Column names must match exactly, including capitalization:

| Column | Type | Options, where it is a Single select |
|---|---|---|
| Job Title | Single line text | |
| Company | Link to another record | Links to `Companies` |
| Posting URL | URL | |
| External ID | Single line text | Used for deduplication. Do not rename. |
| Location | Single line text | |
| Date Found | Date | |
| Status | Single select | `New`, `Scored`, `Applied`, `Interviewing`, `Closed` |
| Fit Score | Single select | `Strong Fit`, `Medium Fit`, `Weak Fit`, `Skip` |
| Recommended Action | Single select | `Apply Now`, `Network First`, `Watchlist`, `Skip` |
| Role Family | Single select | `Strategy / Ops`, `BizOps`, `Analytics`, `GTM`, `PMM`, `Chief of Staff`, `Other` |
| Fit Rationale | Long text | |
| Key Requirements | Long text | |
| Notes | Long text | |

The single-select options also have to match exactly. Airtable rejects a value that is not already an option.

### Step 2: Import the workflow

In n8n, go to Workflows, then Import from File, and select `workflow.json`.

### Step 3: Add your credentials

In n8n under Settings, then Credentials, create two:

- An **Airtable Personal Access Token**, with `data.records:read`, `data.records:write`, and `schema.bases:read` on your base.
- An **Anthropic API** credential holding your API key.

Then open each of the three Airtable nodes and the `Call Claude` node and select the credential you just made. The placeholders in `workflow.json` (`REPLACE_WITH_YOUR_CREDENTIAL_ID`) do not resolve on their own.

### Step 4: Point the workflow at your base

In each Airtable node, replace `REPLACE_WITH_YOUR_BASE_ID` and `REPLACE_WITH_YOUR_TABLE_ID` by picking your base and table from the dropdown.

### Step 5: Set your target companies

Edit the `companies` array at the top of each fetch node. The `token` is the board slug from the company's careers URL.

```js
const companies = [
  { name: 'Example Co', token: 'exampleco' },  // boards.greenhouse.io/exampleco
];
```

The repo ships with placeholder slugs that do not exist. Replace them or every fetch returns nothing.

To find a slug, open a company's careers page and look at the URL:

| Platform | Careers URL | Slug |
|---|---|---|
| Greenhouse | `job-boards.greenhouse.io/gitlab` | `gitlab` |
| Lever | `jobs.lever.co/palantir` | `palantir` |
| Ashby | `jobs.ashbyhq.com/cohere` | `cohere` |

Add the same company names to your Airtable `Companies` table so the link field resolves.

### Step 6: Set your candidate profile

In `Build Prompt`, replace the `Candidate: [YOUR PROFILE ...]` placeholder with a description of yourself and the roles you want. Scoring is relative to this, so be specific about seniority and function.

### Step 7: Tune the filters

`Pre-Filter` holds an `excludeTitles` array and an `allowedLocations` whitelist. Both ship with one person's preferences and are worth rewriting for your own search.

### Step 8: Test before scheduling

Run the workflow manually once. Check that rows appear in Airtable with a Fit Score attached. Only then enable the schedule trigger.

## Design notes

Decisions that mattered more than they look:

- **Cheap filter before expensive filter.** The keyword pre-filter is unglamorous. It removes most postings for free, so the model only ever sees plausible candidates.
- **Batching over per-item calls.** Scoring one job per API call is the obvious implementation. It costs roughly an order of magnitude more. The `Build Prompt` node assembles many jobs into one numbered request.
- **A rigid output contract.** The system prompt demands exactly four prefixed lines per job. Free-form output would make `Parse Scores` unreliable.
- **Drop bad rows instead of writing them.** Anything that fails to parse is marked `Rejected` and never reaches Airtable. Partial records are worse than missing ones.
- **Recommend an action, not just a score.** Every row gets Apply Now, Network First, Watchlist, or Skip. A score alone still leaves you deciding what to do next.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Fetch nodes return zero jobs | The company slugs are still the placeholders, or the slug is wrong. Test one in a browser: `https://boards-api.greenhouse.io/v1/boards/YOUR_SLUG/jobs` |
| Airtable writes fail | A column name or single-select option does not match Step 1 exactly. |
| `Company` field is empty | The company is missing from the `Companies` table, or the name does not match the `name` in the fetch node. |
| Everything scores as Skip | The candidate profile in `Build Prompt` is still the placeholder. |
| Workflow errors on the Claude node | Check the API key and that your Anthropic account has credit. |

## What is in this repo

This is a redacted export. Credential IDs, the Airtable base and table IDs, the original target-company list, and the candidate profile are all placeholders.

No API keys were ever stored in the workflow file. n8n keeps credentials server-side and the export references them by ID only.

## License

MIT
