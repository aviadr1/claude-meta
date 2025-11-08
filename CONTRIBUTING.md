# Contributing to claude-meta

We're building a living playbook for turning `CLAUDE.md` into a self-improving knowledge base. Your feedback, field notes, and experiments keep this repository evolving so future sessions start smarter than the last one.

## Before You Start
- Read the full article in [`README.md`](README.md) to understand the reflection loop and meta-rules.
- Skim the [starter template](CLAUDE_TEMPLATE.md) and [full example](CLAUDE_FULL.md) so new guidance lands in the right section.
- Reproduce your improvements locally before opening a pull request—this repo is all documentation, so screenshots, transcripts, or short writeups go a long way.

## Ways to Level Up the Playbook

### 1. Share Adoption Stories
- Tell us how you adapted the meta-rules for your team or project.
- Include the prompt you used, the mistake that triggered it, and the rule you added to `CLAUDE.md`.
- Highlight measurable impact: fewer review comments, faster onboarding, cleaner diffs, etc.

### 2. Publish Quick-Start Guides
- Create lightweight checklists that help new contributors run the reflection cycle in minutes.
- Map each step to the relevant section of the template so readers can trace examples back to authoritative guidance.
- Link to any shell aliases, shortcuts, or scripts that automate the reflection prompt.

### 3. Automate the Boring Parts
- Contribute helper scripts that open `CLAUDE.md`, append new learnings, or validate that the summary stayed in sync.
- Document editor or terminal integrations that trigger “Reflect, abstract, generalize, add to CLAUDE.md.”
- Show how you wired this workflow into CI, bots, or pre-commit hooks.

### 4. Capture Anti-Patterns and Postmortems
- Document mistakes that slipped through and how the meta-rules evolved to prevent a repeat.
- Use before/after snippets with ✅/❌ callouts when the difference is subtle.
- Add “Warning Signs” bullets only when the meta-section says the nuance warrants it.

## Making a Change
1. Fork the repo and create a topic branch (`doc/your-topic-name`).
2. Update or add markdown files—no code lives here, so clarity and structure beat cleverness.
3. Run a quick spell/format check if you have tooling handy.
4. Submit a PR that summarizes the learning and calls out any new rules so reviewers can verify the summary stayed accurate.

## Feedback Loop
- Open issues when something in the template confused you or when the article should cover a scenario it misses today.
- Start a discussion if you want to explore a new meta-rule before writing it down.
- Drop a star ⭐️ and share success stories—we love hearing how the loop is working in the wild.

Thanks for helping every session start smarter than the one before it.
