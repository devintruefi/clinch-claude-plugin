---
name: setup
description: Set up Clinch from inside Claude. Use when the user runs /clinch:setup, or asks to set up, onboard, populate, or fill in Clinch from their CRM, docs, email, Slack, or call transcripts.
---

# Clinch setup

You are running the Clinch setup workflow from the Clinch plugin, version 1.3.0. When you call complete_setup at the end, pass plugin_version "1.3.0" and surface "cowork" (use "claude_code" if you are running in Claude Code, or "claude_chat" in a regular Claude chat).

If the Clinch tools (get_company_profile, add_competitor, complete_setup) are not available, tell the user to sign in to the Clinch connector that came with this plugin (Settings, Connectors, Clinch, Connect), then run /clinch:setup again. Stop there.

If Clinch already has competitors and a company profile, say so, and only fill gaps the user wants filled.

Follow the workflow below as if the user wrote it.

Goal: every competitor that really shows up in my pipeline is tracked in Clinch, each has rivalry context grounded in my real data, and my company profile is complete. Generic or invented content is worse than leaving a field empty.

Step 1. Confirm scope before doing anything.
- Call get_company_profile and get_competitive_landscape to see what Clinch already has, including my company name and website.
- List the Claude connectors I have that look like a CRM (Salesforce, HubSpot, Pipedrive, Lightfield), docs (Notion, Google Drive, SharePoint), or meeting transcripts (Krisp, Zoom, Gong, Fathom), and ask me which to use.
- Ask if there are competitors I want excluded, and confirm the time window (default: last 90 days of CRM and transcripts, all docs).
- Only read my own email and direct messages, never another person's, and skip anything I say is confidential.

Step 2. Find competitors.
- Pull competitor names from CRM opportunities (competitor fields and notes), transcripts, and competitive or win/loss docs.
- Group them: already tracked, new, and out of scope (excluded, or mentioned once with no context).
- Show me the list and wait for my go-ahead before adding any. Adding competitors uses my plan's competitor limit.

Step 3. Add approved competitors with add_competitor.
- Confirm each homepage with web_fetch. For optional URLs (pricing, careers, changelog, docs), only pass a URL you found as a real link on their site and confirmed loads. Never guess paths like /careers. If unsure, omit it; Clinch finds these pages itself.
- Tell me about any URLs the response lists as dropped. If a tracked competitor's name is misspelled, offer to fix it with update_competitor.

Step 4. Fill rivalry context for each tracked competitor.
- Call get_competitive_context first to see what is already filled.
- Draft loss_reasons, win_reasons, and notes from real evidence: closed lost and closed won notes, competitive call transcripts, and "vs" docs. Quote sources where you can.
- Show me the drafts, then call set_competitive_context after I approve. Use merge_mode "merge" for empty fields and "append" to add to text a teammate already wrote. Never resend existing text.

Step 5. Fill empty company profile fields.
- Use the empty_fields list from get_company_profile, especially why_you_lose and key_differentiators.
- Show me drafts, then call set_company_profile_field after I approve, with merge_mode "merge". If Clinch says only the workspace owner can do this, skip it and tell me.

Step 6. Log closed deals I approve.
- Offer to log recent closed deals from the CRM with log_deal_outcome, including no decision and status quo losses (no competitor needed). Pass crm_deal_id so nothing is logged twice.

Rules for every write:
- Include an evidence argument naming the exact source: a CRM record id, transcript and timestamp, doc link, or "user said so in chat".
- Never invent evidence or reasons. If the data is thin or conflicting, ask me instead of writing.
- I am here approving these writes, so leave review_status at its default.
- Do not create briefs during setup.

Step 7. Finish.
- Call complete_setup with the sources you read (for example "salesforce", "gmail", "gong"), how many competitors you added, how many competitors' context you updated, and how many profile fields you filled.
- Summarize what you added and updated, what you skipped and why, and remind me that battlecards refresh automatically as Clinch collects data.
