# Clinch for Claude

Competitive intelligence from [Clinch](https://www.getclinch.ai) inside Claude. Ask what a competitor changed and when, read the current battlecard, check your team's win and loss record, prepare a one-page brief before a sales call, and log deal outcomes from your conversations. Answers come from your own Clinch workspace, not a web search.

## What it includes

- **The Clinch connector**, a remote MCP server at https://www.getclinch.ai/api/mcp. You sign in with your Clinch account and approve read and write access.
- **/clinch:setup**, a guided setup. It reads the sources you choose (your CRM, documents, email, Slack or call transcripts, through connectors you have already added to Claude), proposes competitors, rivalry notes and company profile text, and writes each one to Clinch only after you approve it.
- **Five skills**:
  - clinch-pre-call-brief: a one-page brief before a competitive call.
  - clinch-post-call-capture: logs a deal outcome or competitor mention after a call.
  - clinch-deal-coach: your record against a competitor and what to say next.
  - clinch-import-from-crm: logs closed deals from the CRM connector you use.
  - clinch-onboard-workspace: fills a new Clinch workspace from your CRM, docs and transcripts.

## Requirements

A Clinch account on the Clinch Pro plan or an active 14 day trial. Sign up at https://www.getclinch.ai/signup.

## What it sends, and where

- Every tool call goes to Clinch at https://www.getclinch.ai/api/mcp with your sign-in token and an x-clinch-plugin-version header carrying this plugin's version number, so Clinch can tell you when a newer version is available. The plugin sends nothing anywhere else.
- Reads return data from your own Clinch workspace. Writes (deal outcomes, competitor mentions, competitors, rivalry notes, company profile fields, battlecard feedback and briefs) are saved to that workspace, are visible to your teammates, and are recorded in an audit log. Claude asks before running the tools that can overwrite or delete data.
- The skills are instructions for Claude. They read your calendar, CRM, documents, email or transcripts only through connectors you added to Claude yourself, and only when you ask. The plugin contains no scripts, hooks or local servers.

## Support

- Email: hello@getclinch.ai
- Documentation: https://www.getclinch.ai/clinch-for-claude
- Privacy policy: https://www.getclinch.ai/privacy
