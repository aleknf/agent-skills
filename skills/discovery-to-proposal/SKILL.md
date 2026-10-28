---
name: discovery-to-proposal
description: Turn presales discovery notes into a requirement matrix, a platform recommendation (Salesforce, Lark, or both), a proposal outline, and a demo script. Use when you have discovery or meeting notes for a prospect and need a requirement matrix, platform recommendation, proposal outline, demo script, or want to go from discovery to proposal. Produces the content behind a proposal, not a branded deck.
---

# Discovery to proposal

Produces the thinking behind a proposal, not the company deck. The consultant moves the content into their own proposal template.

## Inputs and output

- **Sources**: the prospect's project note and its meeting notes, or notes pasted into the conversation. Ask where they live if unclear.
- **Output**: one markdown file, `<Client> - Requirement & Proposal.md`, next to the client's project note (or wherever the user's notes convention says). Follow the existing frontmatter and link conventions of that workspace.

## Rules

- **Every requirement traces to a source**: a meeting note and the line it came from. No requirement without one. Things you infer are listed separately as "Assumptions — confirm with client".
- **No invented numbers**: user counts, license counts, prices, timelines and budgets stay `[fill in]` unless they appear in the source.
- **Platform claims must be true for the current product.** If you are not sure a feature exists in the client's edition (or in Lark's current plan), write "needs verification" instead of asserting it. Check the vendor docs when you can.
- **Shared demo environments**: if demo orgs are shared across clients, never plan org-wide changes there (global picklists, large profiles, triggers on shared objects). For demo builds, prefer declarative solutions (Flow, LWC with standard data modules, configuration) over Apex. If Apex is truly needed, say so and explain the test-coverage consequence before writing any.
- **Not the internal deck**: no slide layout, branding or company boilerplate. Content only.
- **No credentials** in any output, and no commit or push unless the user asks. Use people's names as the user's contact list spells them.

## Steps

1. Read the project note's latest status and changelog, then every meeting note for that client. List what is still unknown.
2. **Requirement matrix** (markdown table):

   | ID | Requirement | Source | Priority (Must/Should/Could) | Area | Salesforce | Lark | Notes / gap |
   |---|---|---|---|---|---|---|---|

   Fit columns use: ✅ standard · ⚙️ config/Flow · 🧩 custom/AppExchange/integration · ❌ not a fit · ❓ needs verification. Area examples: Sales, Service, Marketing, Approval, Collaboration, Reporting, Integration.
3. **Platform recommendation**: Salesforce, Lark, or both, with the reasoning tied to requirement IDs (e.g. "R3, R5, R8 are CRM pipeline + forecasting → Salesforce Sales Cloud"). If both, say who owns what and where the integration sits. Name the risks and what would change the recommendation. Mention existing systems from discovery (ERP, document storage, etc.) and how they connect.
4. **Proposal outline** (headings with 1–3 bullet points each): context and pain points in the client's words, objectives, scope by phase (start with the base modules if the client asked to start small), solution overview, integration, out of scope, assumptions, open questions, next steps. Effort and price sections stay `[fill in]`.
5. **Demo script**: 3–6 scenes, each tied to requirement IDs: persona, what to click or show, the one sentence that lands the value, data to prepare, and the environment to build it in. Add a 2-line opening and close.
6. **Open questions for the next call**, grouped by who can answer them.
7. Write the output file, add a changelog line to the project note linking it (if the workspace keeps one), and report what is still marked `[fill in]`, `❓` or "Assumptions".
