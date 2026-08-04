<div align="center">

<h1>Audit design tokens</h1>

**A Claude Code skill that scans a codebase for design token drift - raw hex colors, off-scale spacing, and near-duplicate values a team stopped routing through its own design system.**

[![CI](https://img.shields.io/github/actions/workflow/status/humbleteam/audit-design-tokens/validate.yml?branch=main&style=for-the-badge&logo=github&label=CI)](https://github.com/humbleteam/audit-design-tokens/actions/workflows/validate.yml)
[![GitHub stars](https://img.shields.io/github/stars/humbleteam/audit-design-tokens?style=for-the-badge&logo=github&color=181717)](https://github.com/humbleteam/audit-design-tokens/stargazers)
[![Last commit](https://img.shields.io/github/last-commit/humbleteam/audit-design-tokens?style=for-the-badge&color=339933)](https://github.com/humbleteam/audit-design-tokens/commits/main)
[![License](https://img.shields.io/badge/license-MIT-blue?style=for-the-badge)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-skill-D97757?style=for-the-badge&logo=anthropic&logoColor=white)](https://github.com/humbleteam/audit-design-tokens/blob/main/SKILL.md)

</div>

Every design system starts clean and drifts. Someone ships a one-off `#2563EB` instead of
`var(--color-primary)`, a `padding: 13px` sneaks past the 4px scale, and nobody can say with
confidence which colors are actually tokens anymore. This skill scans a codebase, finds
every raw value that should have been a token, and returns a severity-ranked report with a
file:line for each finding and a smallest-diff-first plan to fix it.

## Table of contents

- [What it does](#what-it-does)
- [Quick start](#quick-start)
- [Usage](#usage)
- [Example output](#example-output)
- [How it works](#how-it-works)
- [How is this different from just asking the model?](#how-is-this-different-from-just-asking-the-model)
- [FAQ](#faq)
- [Related skills](#related-skills)
- [Who maintains this](#who-maintains-this)

## What it does

- Finds token definitions in CSS custom properties, `tailwind.config.js`, a dedicated
  `tokens.json` / `theme.ts` file, or a styled-components theme object - whichever the
  codebase actually uses.
- Flags raw color literals (`#hex`, `rgb()`, `hsl()`) outside those definitions, with the
  file and line number of each one.
- Flags spacing values that fall off the codebase's own scale (a `padding: 13px` on a 4px
  grid, for example).
- Groups near-duplicate colors by hue and lightness so you can see merge candidates at a
  glance instead of hunting for them by eye.
- Counts distinct font sizes, radius values, shadow depths, and raw `z-index` numbers -
  sprawl in any of those is as real a drift signal as color.
- Ranks every finding P0 (breaks theming), P1 (scale drift), or P2 (consolidation candidate)
  and ends with an ordered, independently shippable fix plan.

## Quick start

**Personal (all your projects):**

```bash
git clone https://github.com/humbleteam/audit-design-tokens ~/.claude/skills/audit-design-tokens
```

**Project-scoped (this repo only, shared with your team via git):**

```bash
git clone https://github.com/humbleteam/audit-design-tokens .claude/skills/audit-design-tokens
```

**Other agents (Cursor, Codex, or any LLM agent):** this skill is plain markdown following
the Agent Skills format. Paste the contents of `SKILL.md` into the agent's system prompt or
custom-instructions field.

After installing, restart Claude Code and confirm the skill loaded - it should appear when
you run `/skills` or ask about design token drift. Claude Code loads skills from
`~/.claude/skills/` (personal) and `.claude/skills/` (project).

## Usage

- "Audit our design tokens" - scans the repo (or the path you name), finds the token
  definitions, and reports every raw value that bypasses them.
- "Find hardcoded colors in src/components" - narrows the scan to one directory and returns
  file:line for every literal color found.
- "Are we actually using our Tailwind theme?" - compares every class in the codebase against
  `tailwind.config.js`, flagging arbitrary-value classes like `bg-[#2563eb]` as drift.

## Example output

The block below is an illustrative example - a sample audit of a fictional codebase, not a
real client or a real repo.

```markdown
# Design token audit - src/

## Summary
- Token sources found: CSS custom properties (22 tokens), tailwind.config.js theme (6 colors)
- Raw colors outside tokens: 14
- Off-scale spacing values: 9
- Near-duplicate color pairs: 3
- Distinct font sizes in use: 12 (target: 8-10 or fewer)
- Radius variants: 5 · Shadow variants: 4
- Hardcoded z-index values: 7

## P0 - breaks theming

| File:line | Value | Nearest token | Suggested fix |
|---|---|---|---|
| src/components/Button.tsx:42 | #2563EB | --color-primary | replace with var(--color-primary) |
| src/components/Modal.tsx:88 | rgba(0,0,0,0.4) | --color-overlay | replace with var(--color-overlay) |

## P1 - scale drift

| File:line | Value | Nearest token | Suggested fix |
|---|---|---|---|
| src/components/Card.tsx:15 | padding: 13px | --space-3 (12px) | round to nearest scale step |

## Consolidation plan
1. Replace 3 raw #2563EB literals in Button.tsx with the existing primary token.
2. Replace the overlay rgba() literal in Modal.tsx with --color-overlay.
3. Round the 9 off-scale spacing values to the nearest 4px step, one component at a time.
```

## How it works

- **Token source first.** The skill locates the codebase's actual token definitions before
  scanning anything - CSS custom properties, a Tailwind theme, a tokens file, or a
  styled-components theme object. Nothing counts as drift until there is a known-good value
  to drift from.
- **File:line, not vibes.** Every finding traces to a real location the skill scanned. Nothing
  in the report is invented or estimated.
- **Category by category.** Colors, spacing, near-duplicates, font sizes, radii, shadows, and
  z-index are scanned and reported separately, since each has a different fix and owner.
- **Severity by damage, not by count.** A single hardcoded color on a hover state outranks
  fifty off-scale spacing values, because the color bug is user-visible today.
- **Renames are proposals, never actions.** If a token's name no longer matches how it is
  used, the skill lists the rename separately for a human to approve.
- **Smallest diff first.** The consolidation plan orders fixes so each one ships as its own
  small, reviewable change, not one giant sweep across the codebase.

## How is this different from just asking the model?

A bare "find hardcoded colors in my codebase" prompt gets a handful of examples from
whatever files the model opens first, with no severity ranking and no sense of the actual
token scale. This skill forces the token-source step before scanning, so every finding is
checked against real values instead of guessed at. It also fixes the report shape - summary
counts, three severity tiers, file:line evidence, a separate rename proposal - so the output
is consistent enough to paste straight into a ticket or a PR description.

## FAQ

**How do I find hardcoded colors in my codebase?**
Ask Claude Code to audit design tokens with this skill installed. It locates your token
definitions first, then scans CSS, SCSS, JSX/TSX, Vue, Svelte, and inline styles for any
color literal that does not match one, returning a file:line for each.

**What is design token drift?**
Drift is any value in the codebase that should route through a design token but does not -
a raw `#2563EB` instead of `var(--color-primary)`. It accumulates through one-off fixes and
copy-pasted components.

**How do I migrate to design tokens?**
Run this skill first to see the size of the problem, then work through the consolidation
plan it returns - ordered smallest-diff-first so each step is an independently shippable PR.

**Does this skill fix the code automatically?**
No, not by default. It returns a report and a proposed plan, and only edits files if you
explicitly ask it to apply a specific step after reviewing the report.

**What if my codebase has no design tokens at all?**
The skill stops and tells you rather than inventing a baseline to audit against. Use
[extract-design-tokens](https://github.com/humbleteam/extract-design-tokens) first to pull a
starting token set from a URL, screenshot, or the codebase's own most common values.

**Does this work with Tailwind?**
Yes. It compares class usage against `tailwind.config.js`'s theme and treats
arbitrary-value classes like `bg-[#2563eb]` or `p-[13px]` as the drift signal, since those
are exactly the places a value bypassed the theme.

## Related skills

Part of a 10-skill open-source kit for design teams by Humbleteam.

- [design-review](https://github.com/humbleteam/design-review) - structured UX critique with a 0-4 score, Before/After/Why fixes, and a citation for every claim.
- [ascii-wireframes](https://github.com/humbleteam/ascii-wireframes) - three distinct layout hypotheses as ASCII wireframes before any hi-fi work.
- [html-mockup](https://github.com/humbleteam/html-mockup) - census-first HTML mockups that match a reference screenshot: exact palette, item counts, component states.
- [extract-design-tokens](https://github.com/humbleteam/extract-design-tokens) - pull palette, type, spacing, radii, and shadows from a URL or screenshot into CSS variables and JSON.
- [design-qa](https://github.com/humbleteam/design-qa) - a pre-ship design QA gate: states, contrast, touch targets, breakpoints, keyboard paths.
- [design-handoff](https://github.com/humbleteam/design-handoff) - turn a finished mockup into a dev-ready spec: tokens, states, accessibility annotations, open questions.
- [accessibility-audit](https://github.com/humbleteam/accessibility-audit) - WCAG 2.2-grounded accessibility review with success-criterion citations and severity levels.
- [ux-writing](https://github.com/humbleteam/ux-writing) - interface copy that reads human: plain-verb microcopy rules and an AI-tell strip pass.
- [design-brief](https://github.com/humbleteam/design-brief) - extract a 5-bullet design brief from messy project inputs, with a gap report for what is missing.

## Who maintains this

[Humbleteam](https://humbleteam.com/) is a digital product design and AI-engineering studio: founded in 2017, working from Prague and Dubai, with 80+ digital awards to the name, including 14 Awwwards wins, a Webby, and a Red Dot. We design digital products for startups and enterprises in fintech, healthtech, sports, and AI, and we build AI infrastructure for design teams - agents, workflows, and skills like this one.

This skill is distilled from the internal playbooks we run on client work: the same checklists behind the case studies at [humbleteam.com/work](https://humbleteam.com/work), for clients like Tinder and Acronis.

- The full 10-skill kit: [Related skills](#related-skills) above, or all repos at [github.com/humbleteam](https://github.com/humbleteam)
- What we do with AI for design teams: [humbleteam.com/ai](https://humbleteam.com/ai)
- Design and AI writing: [humbleteam.com/blog](https://humbleteam.com/blog)
- LinkedIn: [linkedin.com/company/humbleteam](https://www.linkedin.com/company/humbleteam/)
- Talk to us: [hi@humbleteam.com](mailto:hi@humbleteam.com)

Issues and PRs welcome.

MIT - see [LICENSE](LICENSE).
