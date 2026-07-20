---
name: bronson-engineering-workflow
description: Use when starting new engineering work at Bronson AI, scoping a Linear ticket, referencing work in a repository, deciding where documentation belongs, or asking about the team's toolchain. Covers the Linear -> repo -> Notion workflow.
---

# Bronson AI — Engineering Workflow

This skill governs how engineering work gets scoped, built, and documented at Bronson AI.
The Linear, GitHub, and Notion MCP servers bundled with this plugin are the tools to use
for each step below — don't ask the user to paste ticket content or doc content that these
tools can fetch directly.

## 1. Scope new work into a Linear ticket first

Before writing any code, confirm a Linear ticket exists in the **Bronson AI** workspace
that describes the work. If one doesn't exist, create it before proceeding.

- If the user describes work verbally ("let's build X", "fix the Y bug"), draft the ticket
  content and confirm with the user before creating it — don't silently create tickets.
- Priority is an integer field: `2` = High, `3` = Medium. Default to Medium unless the user
  indicates urgency.
- Team: Bronson

Treat the ticket as the source of truth for scope. If a request conflicts with or expands
beyond an existing ticket's description, flag the discrepancy rather than silently expanding
scope.

## 2. Reference the ticket in all repo work

Every branch, commit, and PR tied to a piece of work must reference its Linear ticket ID.

- Branch names: Follow the convention: `<ticket-id>-short-description` ex. bronson-123-fix-pagination-bug
- Commit messages: include the ticket ID, e.g. `[BRONSON-123] Fix pagination bug`
- PR descriptions: link the Linear ticket explicitly
- Pull the ticket via the Linear MCP server as the working definition of scope — don't
  infer requirements that aren't in the ticket.
- Always use branches and NEVER commit directly to `main` or `master`. Use PRs for all changes, even small ones.

## 3. Where documentation belongs

Markdown in the repo is fine for anything scoped to the code itself: folder-level READMEs,
setup instructions, inline explanations of how a module works, CONTRIBUTING-style notes.
Write these as normal `.md` files alongside the code they describe.

High-level decision-making and diagrams go in Notion instead: design docs, architecture
decisions, vendor/toolchain evaluations, runbooks, and anything a teammate would need to
find without already knowing which repo to look in.

- Use the Notion MCP server to create or update Notion pages directly.
- Docs live in the **Bronson AI** team space, under the subfolder matching the relevant
  repository. If a repo doesn't have a subfolder yet, create one before adding the page.
- If unsure which side something falls on, ask: does this explain *how the code works* (repo
  markdown) or *why a decision was made* (Notion)?

## 4. Toolchain

The following are connected by default through this plugin — don't suggest alternatives
unless the user asks to evaluate a change:

- **Linear** — ticket tracking and scope of record
- **GitHub** — code hosting, PRs, reviews
- **Notion** — documentation and decision records
- **Opik** — LLM evaluation and regression testing for FOIAger; used for a pre-deployment
  pipeline gated on metric floors and baseline comparisons
