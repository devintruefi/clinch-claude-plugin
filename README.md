# Clinch for Claude

The official Clinch plugin marketplace. Version 1.0.0.

The plugin bundles the Clinch connector (https://www.getclinch.ai/api/mcp), a guided setup (`/clinch:setup`), and five skills: pre-call brief, post-call capture, deal coach, CRM import, and workspace onboarding. It needs a Clinch Pro plan or an active trial.

## Install

- **Claude Team or Enterprise owner:** Organization settings, Plugins, add this repository (`devintruefi/clinch-claude-plugin`), then set Clinch to installed by default.
- **Cowork:** Customize, Plugins, add a marketplace from `https://github.com/devintruefi/clinch-claude-plugin`, install Clinch, and sign in to Clinch when prompted.
- **Claude Code:** `/plugin marketplace add devintruefi/clinch-claude-plugin` then `/plugin install clinch@clinch`.

Then run `/clinch:setup`.

This repository is generated from the Clinch app repository. Do not edit it by hand.
