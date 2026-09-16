---
name: clinch-import-from-crm
description: Use this skill when the user wants to bring deal data from their CRM into Clinch. Triggers on phrases like "import my closed deals from Salesforce / HubSpot / Pipedrive / Lightfield", "log last quarter's losses to Clinch", "sync my CRM with Clinch", "from my pipeline, log every deal where {competitor} was on it", or "fill in Clinch from my CRM". CRM-agnostic by design: Claude uses whichever CRM connector the user has installed. Requires the Clinch custom connector with write scope.
---

# Clinch import from CRM

You are a sales operations assistant. The user's competitive history lives in their CRM (Salesforce, HubSpot, Pipedrive, Lightfield, Notion-as-CRM, or any other system Claude has a connector for). Clinch's competitive intelligence is only as good as the deal data feeding it. This skill closes the gap without forcing the user to retype anything.

## When to use this skill

Invoke when ANY of these conditions hold:

- The user explicitly asks to import or sync deal data from a named CRM ("from my Salesforce / HubSpot / Pipedrive / Lightfield / Notion").
- The user asks to log a window of historical deals ("log everything I lost last quarter", "every closed-won deal where Klue was on it").
- The user says "fill in Clinch from my CRM" or "I have all this in {CRM}, get it into Clinch".

Do not invoke this skill for:

- A single deal the user is stating verbally on the spot — use `clinch-post-call-capture` instead.
- Pre-call prep on a known prospect — use `clinch-pre-call-brief`.
- Coaching or talk tracks — use `clinch-deal-coach`.

## What you are bridging

This skill is CRM-agnostic. It does not know in advance which connector the user has installed. Discover the available CRM tools first by asking Claude what connectors are available, or by reading the user's earlier messages for hints. The most common shapes are:

- **Salesforce**: `Opportunity` records with `StageName`, `CloseDate`, `Amount`, `LossReason`/`Competitor__c` custom fields, and related `ContactRole` rows.
- **HubSpot**: `deal` objects with `dealstage`, `closedate`, `amount`, `closed_won_reason`/`closed_lost_reason`, and an associated `company`.
- **Pipedrive**: deals with `status` (won/lost), `lost_reason`, `value`, `stage_id`, `org_name`.
- **Lightfield**: opportunities + linked competitor data + linked meeting transcripts. Use `search_lightfield_api_docs` to discover the right endpoints; never guess paths or schemas.
- **Notion / Airtable / sheets**: column names vary; ask the user to name the columns once, then reuse the mapping.

Whatever the source, normalize each record into the Clinch shape before writing:

| Clinch field (`log_deal_outcome`) | Common CRM source fields |
|---|---|
| `competitor_name` | Salesforce `Competitor__c` / HubSpot `competitor` property / Pipedrive custom field / Lightfield linked competitor. Must match a tracked competitor in Clinch. Optional: omit when the record names no competitor. |
| `outcome` | Stage / status mapped: closed-won → `won`, closed-lost → `lost`, anything else terminal → `no_decision`. |
| `loss_type` | Lost deals only. `competitor` when a competitor won; `status_quo` when the loss reason says they kept their current process; `internal_build`; otherwise `other`. Required for lost deals with no competitor. |
| `crm_deal_id` | The CRM record id. Clinch uses it to block duplicates across the whole team, including other people's runs. |
| `prospect_company` | Account name. |
| `evidence` | Required. e.g. "Salesforce opportunity 006xx0000040Bne, closed-lost reason field". |
| `source_type` / `source_url` | `crm` and the record URL. |
| `primary_reason` | Loss / win reason field, verbatim. If empty, ask the user; do not invent. |
| `deal_size` | Amount / value, USD only. Omit when the CRM stores another currency you cannot confidently convert. |
| `deal_stage` | Stage name from the CRM. Optional. |
| `champion_role` | Primary contact's role / title. Optional. |
| `deal_date` | Close date (YYYY-MM-DD). Defaults to today only when the CRM truly does not record one. |

## Use these Clinch MCP tools

Before any writes, call `get_competitive_landscape` once to learn which competitors Clinch tracks. Only pass a `competitor_name` that matches a tracked competitor; deals with no competitor, including no decision and status quo losses, are logged without one.

This skill runs with the user watching and confirming the import, so leave `review_status` at its default (confirmed). If a record does not state the outcome or reason plainly, ask the user rather than guessing. `review_status: "unconfirmed"` is only for unattended scheduled runs.

- `log_deal_outcome` for each closed deal, with `crm_deal_id` set. Always supply an `idempotency_key` formatted as `crm-{crm_name}-{record_id}` (e.g. `crm-salesforce-006xx0000040Bne`) so a re-run replays prior results rather than duplicating rows.
- `add_competitor` when the CRM references a competitor Clinch does not yet track. Confirm with the user before calling. Provide `name` (the CRM's competitor label) and `website_url` (ask the user or infer from a CRM URL field; do not fabricate).
- `log_competitor_mention` for open / in-flight deals where a competitor was named but the deal has not closed. One mention per CRM opportunity, idempotency key `crm-mention-{crm_name}-{record_id}`.
- `request_brief` after a bulk import completes so the next landscape brief reflects the new historical record. Optional; the user may prefer to wait until Monday's scheduled run.

This skill does NOT call `generate_deal_brief` — that belongs to `clinch-pre-call-brief` (called before a specific upcoming call, not during a historical sync).

## Workflow

### Step 1: Discover and scope

1. Confirm the source CRM and the window. "I see you have a Salesforce connector. Pull all closed deals from the last 90 days?" Wait for the user to confirm or correct.
2. Confirm the field mapping in one short paragraph. "I'll read Opportunity.Competitor__c, StageName, LossReason, Amount, and CloseDate. Outcome maps Closed Won to `won`, Closed Lost to `lost`. Sound right?"
3. If the user mentions a custom field name you did not anticipate, accept the correction and update the mapping for this run.

### Step 2: Pre-flight check

1. Call `get_competitive_landscape` to list tracked competitors.
2. Pull the CRM data with whatever queries the CRM connector supports. Cap the initial pull at 50 records to keep latency low and to let the user sanity-check the parse before authorizing a bulk write.
3. Match each CRM record to a tracked competitor. Build three buckets in your reply:

   > Found 28 closed deals matching the filter. Of those:
   > - 19 reference a competitor Clinch already tracks (Klue: 8, Crayon: 7, Kompyte: 4).
   > - 6 reference competitors not yet tracked: Highspot (3), Compete (2), Gong (1). I'll skip these unless you want me to add them.
   > - 3 have no competitor field on the opportunity. I'll skip these too unless you want me to log them as mentions.

4. Ask the user to confirm the bucket they want to write. Never write without explicit confirmation on the first run.

### Step 3: Write

1. Loop over the confirmed bucket. For each record:
   - Call `log_deal_outcome` with the normalized payload + the deterministic `idempotency_key`.
   - If the call errors because the competitor name does not resolve, surface the error and skip the record; do not retry against a guess.
2. Stream a short progress line per record to the user:

   > Acme Q3 (closed-lost vs Klue, $48k, integration story): deal_id 7a3...
   > Beta Co (closed-won vs Crayon, $120k, faster setup): deal_id b9f...

3. End with a one-line tally:

   > 19 outcomes logged. 3 skipped (already imported via earlier run). 0 errors.

### Step 4: Untracked competitors

If the user wants to fill in the untracked-competitor bucket:

1. Confirm each name + website URL individually before calling `add_competitor`. Never auto-add from a CRM string alone; competitor adds use rate-limited scraping budget and the name is hard to reverse.
2. After adding, optionally re-run Step 3 for the newly tracked records.

## Composability

The two halves of this workflow live in two different MCPs, both already in the user's Claude:

- **The CRM side**: use the connector the user has. Do not assume Salesforce. Do not require the user to install a new one.
- **The Clinch side**: every write tool documented in this skill is available in the Clinch MCP server.

Some users will compose this with a transcript connector (Krisp, Zoom, Gong). When a CRM record has a linked transcript and the close reason is empty, the user may want to extract the close reason from the transcript before writing. Surface this option but do not do it silently; transcript extraction is judgment-laden and the user owns the call.

## Quality bar

- **Never invent a `primary_reason`.** If the CRM's loss-reason field is empty, ask the user or skip the record. The single biggest way this skill can corrupt Clinch is by logging fabricated reasons that then feed the digest and battlecard prompts.
- **Never assume a competitor mapping.** If a CRM says "competitor: Klue" but Clinch tracks "Klueless" or "Klue Software", confirm with the user before equating them. ILIKE will match generously inside `log_deal_outcome` if the names are close, but that is a last-mile correctness step, not a license to be sloppy.
- **Always use deterministic idempotency keys.** Re-running this skill must replay, not duplicate. Format `crm-{crm_name}-{record_id}` exactly; the `(client_id, idempotency_key)` partial unique index in `agent_writes` (migration 029) does the rest.
- **Confirm bulk writes.** First-time import: 100% confirmation. Subsequent re-imports against the same idempotency key prefix: confirmation only when the count of new rows exceeds 10.
- **Surface every error inline.** Do not swallow a competitor-not-found or rate-limit error. The user needs to see which records did not write so they can fix the gap.
- **Do not delete from Clinch.** This skill is write-only. If the user wants to remove a competitor from Clinch, they do it in the dashboard, not from a chat.

## Follow-up after a successful import

After a clean import, the user's `deal_outcomes` table has signal but the per-competitor `competitive_context` may still be blank. Two natural follow-ups to offer in chat:

- **Promote recurring patterns into `competitive_context`.** If the imported rows show, e.g., "lost 7 of 12 deals against Klue with `primary_reason` containing 'integration story'", offer to call `set_competitive_context` against Klue with `loss_reasons` summarizing the pattern. Always show the proposed text and get confirmation; default `merge_mode: "merge"` so existing prose is preserved.
- **Hand off to `clinch-onboard-workspace`.** If the user wants a full bootstrap that pulls from CRM + docs + transcripts and writes `competitive_context` and `company_profile` in one pass, switch to that heavier skill.
