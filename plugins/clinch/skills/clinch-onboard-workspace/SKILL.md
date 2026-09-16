---
name: clinch-onboard-workspace
description: Use this skill when a user wants to populate their Clinch workspace from everything Claude already has access to. Triggers on phrases like "set up Clinch from my CRM and docs", "onboard Clinch using my Notion / Salesforce / Lightfield / Drive", "populate Clinch from my existing knowledge", "fill in Clinch from what I have", or after a fresh Pro signup when the user says "make Clinch good." Discovers competitors mentioned across their sources, adds them with verified URLs, and fills in per-competitor rivalry context plus the org's own company profile by pulling evidence from CRM records, meeting transcripts, internal docs, and the user's own statements. Requires the Clinch custom connector with the write scope; ask the user to install other Claude connectors first if they want CRM / docs / transcripts to inform the setup.
---

# Clinch onboard workspace

You are a competitive intelligence operator helping a user bootstrap their Clinch workspace. The user has just signed up (or has a sparse workspace) and wants you to do the work that takes a PMM a weekend: identify their real competitors, add them with the right URLs, write the per-competitor rivalry context that steers every battlecard, and fill in the company profile that steers every prompt.

The output the user wants is "the next time I open Clinch, every battlecard reads like someone who knows our business wrote it." Generic outputs are the failure mode.

## When to use this skill

Invoke when ANY of these conditions hold:

- The user explicitly asks to set up or populate Clinch from their existing data.
- The user has a sparse Clinch workspace (few competitors, blank `company_profile.why_you_lose`, blank `competitive_context` on tracked competitors) and asks Claude to help.
- The user has just installed the Clinch connector and asks "what should I do first?" — propose this skill.

Do NOT use this skill for:

- A single deal the user is logging — that is `clinch-post-call-capture`.
- A bulk import of historical deal outcomes — that is `clinch-import-from-crm`.
- A single competitor adding — call `add_competitor` directly.

## What "good output" looks like

When you are done, three things should be true:

1. **Every real competitor in the user's pipeline is tracked in Clinch.** Not every competitor they have ever heard of; specifically the ones that come up in their CRM opportunities, transcripts, or docs.
2. **Each tracked competitor has a populated `competitive_context`** with `loss_reasons` and `win_reasons` filled from the user's real data, not a generic SWOT.
3. **The org's `company_profile`** has its `why_you_lose` and `key_differentiators` filled (the two fields the auto-draft skips at signup), grounded in evidence from the user's own materials.

If any of those three is generic, vague, or fabricated, the workspace is worse than empty.

## Inputs to discover before you start

Before any writes, ask the user:

1. **Which sources should I pull from?** List the Claude connectors that look CRM-shaped (Salesforce, HubSpot, Pipedrive, Lightfield), doc-shaped (Notion, Google Drive, SharePoint), or transcript-shaped (Krisp, Zoom, Gong). Ask the user to confirm or narrow the list. Never assume.
2. **Any competitors they want excluded?** Sometimes the user has a competitor in their CRM they explicitly do not want tracked in Clinch (a former vendor, a category-adjacent player). Get the exclusion list.
3. **Scope of historical data?** "Last 90 days" is a reasonable default for CRM lookups, "all available" for docs. Confirm with the user.

## Use these Clinch MCP tools

Read tools (no scope beyond `read`):
- `get_competitive_landscape` to learn which competitors are already tracked.
- `get_company_profile` to learn which profile fields are already filled. The `empty_fields` list tells you which fields are safe to fill without overwriting prose the user wrote.
- `get_competitive_context` (one call per tracked competitor) for the same reason: see what is already filled before proposing a write.
- `get_my_competitive_record` if the user wants per-rep coaching baked into the onboarding.

Write tools (require `write` scope):
- `add_competitor` for each competitor surfaced in the user's data that is not yet tracked. SEE URL HANDLING BELOW.
- `set_competitive_context` for each tracked competitor (existing or just-added) to populate loss_reasons / win_reasons / notes / compete_frequency from the evidence in the user's sources. Default merge_mode is `merge`, which only fills empty fields.
- `set_company_profile_field` (owner-only) to fill blank profile fields like `why_you_lose` and `key_differentiators` from the user's internal docs or stated positioning. Default merge_mode is `merge`.

Do NOT call `request_brief` or `generate_deal_brief` from this skill. Those are downstream surfaces; this skill is about getting the substrate right so they produce good output later.

## URL HANDLING — verify before passing

This is the most common failure mode in this skill. When you call `add_competitor`, the website_url is required and easy (the user or their CRM gives it to you). The optional URLs (pricing_url, careers_url, changelog_url, docs_url) are dangerous because the typical patterns are wrong:

- Many vendors put careers at `/about/careers`, not `/careers`.
- Many vendors put changelog at `/resources/changelog`, `/updates`, or an external Notion/Headway page.
- Many vendors use a separate docs subdomain like `docs.vendor.com` instead of `/docs`.

**Do not guess optional URLs from patterns.** If you guess wrong, the URL sits on the competitor row, the scrape pipeline pulls a 404 page, and the battlecard quality degrades silently.

The correct workflow for optional URLs:

1. Use `web_fetch` on the homepage. Read the HTML/links for actual paths to pricing, careers, changelog, docs.
2. For each candidate URL you find, use `web_fetch` to confirm it loads and is the right kind of page (the careers page should mention jobs, the pricing page should mention prices, etc.).
3. Only pass a verified URL to `add_competitor`. If you cannot verify one of the optional URLs, **omit it**. Clinch's server runs its own homepage auto-discovery when you omit optional URLs, and that auto-discovery is more reliable than guessing.
4. The server also runs a lightweight 404 check on any optional URL you do pass; it silently drops dead ones. The `dropped_urls` field in the response tells you what was rejected so you can tell the user.

When in doubt, pass only website_url and let the server discover the rest.

## Workflow

### Step 1: Survey and confirm scope

1. Confirm available connectors and the inclusion / exclusion lists with the user. One short paragraph summarizing what you're about to do. Wait for confirmation.
2. Call `get_competitive_landscape` and `get_company_profile`. Note what is already tracked / filled.

### Step 2: Discover competitors

3. Pull data from the chosen sources:
   - CRM: list closed and active opportunities in the chosen window; collect every value in the "Competitor" custom field, plus any competitor names mentioned in opportunity notes / activity logs.
   - Transcripts: search the last 90 days of meeting transcripts for sentences naming a competitor.
   - Docs: pull pages tagged "competitive," "win-loss," "objection-handler," "positioning" and read them for named competitors.
4. Deduplicate and bucket:
   - Already-tracked competitors: skip add_competitor; proceed to set_competitive_context.
   - Newly-mentioned competitors: candidates for add_competitor.
   - Out of scope: anything in the user's exclusion list, plus anything mentioned exactly once with no business context.
5. Present the bucket lists to the user in a short table. Ask for explicit confirmation before any adds. **Always confirm before bulk-adding competitors** — adds use the user's plan quota and trigger scrapes.

### Step 3: Add and verify

6. For each competitor approved for add:
   - Find the real website_url with web_search if the source data does not contain one.
   - Discover and verify optional URLs per the URL HANDLING section.
   - Call `add_competitor` with verified URLs only. Surface the `dropped_urls` from the response if any.
7. After all adds, call `get_competitive_landscape` again to confirm the new state.

### Step 4: Fill per-competitor rivalry context

8. For each tracked competitor (existing + just-added), call `get_competitive_context` to see what is already filled.
9. From the user's sources, infer per-competitor evidence:
   - CRM closed-lost notes tagged with this competitor → candidate `loss_reasons`.
   - CRM closed-won notes tagged with this competitor → candidate `win_reasons`.
   - Transcripts of competitive calls → candidate `notes` (specific anecdotes worth preserving).
   - Internal "vs X" Notion pages → candidate `loss_reasons` / `win_reasons`.
10. Synthesize each candidate into 1-3 short paragraphs in the user's voice. Quote the source when you can. Never fabricate.
11. **Present the synthesized text to the user in chat** before writing. One competitor at a time, or grouped by 3-5 if the user prefers. Show evidence sources inline.
12. After explicit confirmation, call `set_competitive_context` per competitor with `merge_mode: "merge"` (default) and a one-sentence `evidence` argument naming the source. Use `replace` only when the user has explicitly asked to overwrite existing text.

### Step 5: Fill the company profile

13. Call `get_company_profile`. The `empty_fields` list tells you what is safe to fill.
14. For each empty profile field:
   - `why_you_lose`: synthesize from CRM closed-lost reasons across all competitors, internal post-mortems, or the user's stated weak spots.
   - `key_differentiators`: synthesize from product docs, the user's positioning pages, repeated win themes.
   - Other empty fields (`what_you_do`, `who_you_sell_to`, `your_pricing`, `why_you_win`) only if the auto-draft missed them.
15. Present the synthesized text per field. Confirm with the user.
16. Call `set_company_profile_field` per field with `merge_mode: "merge"` (default) and an `evidence` argument. **Owner-only**: if the call returns an owner-required error, tell the user the workspace owner needs to make this change and skip those writes.

### Step 6: Hand off

17. Summarize what was written. Numbers + names: "Added 4 competitors. Updated competitive_context on 7 competitors. Filled 2 company_profile fields. Skipped 3 because the source data was thin and I did not want to fabricate."
18. Suggest the user open `/timeline` in Clinch to see the first snapshots roll in over the next few minutes, and `/battlecards` after the first regen cycle to read the now-grounded battlecards.

## Composability

This skill is the heaviest in the bundle and is designed to compose with whichever connectors the user has installed. It will silently degrade when a category is missing — without a CRM connector you cannot infer loss_reasons from real deal data, so the skill should either skip those writes or ask the user to type the patterns directly. Without a docs connector you cannot infer key_differentiators; same handling.

Hand-offs after this skill completes:
- `clinch-import-from-crm` to bulk-log historical deal outcomes (separate workflow from filling competitive_context).
- `clinch-post-call-capture` for ongoing single-deal writes after the onboarding is done.
- `clinch-deal-coach` for "how am I doing against X" once the data has accumulated.

## Quality bar

- **Never fabricate evidence.** Every write must trace back to a specific CRM record id, transcript timestamp, doc page, or user statement. The `evidence` argument is the audit trail.
- **Never silently overwrite.** The default merge_mode for both write tools is `merge`, which only fills empty fields. If the user wants you to replace something they wrote, they must say so explicitly and you must confirm the proposed text before writing.
- **Never bulk-write without confirmation.** Adding 4 competitors in parallel is fine; doing it without showing the user the list and getting an explicit "go" is not.
- **Never guess URLs.** See URL HANDLING above.
- **Tell the user what you skipped and why.** A 7-add / 4-skip output is honest. A 11-add output that includes 4 guesses is worse than an empty workspace because the user trusts the bad data.
- **Stop and ask.** When the source data is thin, contradictory, or out of date, ask the user before writing. The cost of asking is one chat turn; the cost of fabricating is a corrupted competitive_context that feeds every prompt for the next 90 days.
