# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code plugin (`postzen-linkedin`) made of twelve skills (`skills/li-*/SKILL.md`) for running a LinkedIn account, plus the PostZen MCP server for publishing. It is mostly prose: the skills are prompt documents, and the only executable code is two dependency-free Python scripts in `skills/li-human/`. It is a fork of Jake Schincariol's linkedin-agent-skill (MIT); PostZen added `/li-publish` and the publishing hand-offs in the other skills.

There is no build, lint config or test suite. The plugin is installed with `/plugin marketplace add postzen-dev/linkedin-agent-skill` then `/plugin install postzen-linkedin`, or by copying `skills/li-*` into `~/.claude/skills/`.

## Commands

The humanizer scripts are stdlib-only Python 3 and resolve `slop.json` relative to their own location, so they can be run from anywhere:

```bash
python3 skills/li-human/humanize.py draft.txt --report   # clean text to stdout, change report to stderr
python3 skills/li-human/humanize.py draft.txt -o clean.txt
python3 skills/li-human/humanize.py - --json < draft.txt  # {text, report}
python3 skills/li-human/detect.py draft.txt               # five-check score panel
python3 skills/li-human/detect.py before.txt after.txt    # before/after delta
python3 skills/li-human/detect.py draft.txt --json
```

Both accept `--lexicon PATH` to use an alternate `slop.json`. `detect.py` exits 0 only on a PASS verdict (overall >= 70 and every check >= 55), 1 otherwise; REVIEW is overall >= 50, below that FLAGGED. The overall score is 60% mean + 40% weakest check.

To sanity-check a change, pipe a deliberately sloppy sample through `humanize.py --report` and then `detect.py before after`.

## Architecture

**Skill pipeline.** Each skill is self-contained (YAML frontmatter `name` + `description` drives when Claude triggers it), but they chain by name:
- Writing skills (`li-post`, `li-comment`, `li-dm`, `li-carousel`, `li-profile`, `li-repurpose`) run their output through `/li-human` before showing it to the user.
- `/li-publish` is the **only** skill that writes to LinkedIn. Other skills end with a copy-ready block and offer `/li-publish` (or `li-plan`/`li-repurpose` use it to schedule). Nothing is posted without an explicit user yes showing text, account, visibility and time.
- `li-publish` calls PostZen MCP tools (`listAccounts`, `listProfiles`, `createPost`, `listPosts`, `listQueueSlots`, `getBestTimeToPost`) and reads back post status after `createPost` rather than trusting the success message.

**User state lives outside the repo**, in `~/.claude/linkedin/`:
- `voice.md`: the user's voice profile (template in `templates/voice.md`); read by every skill.
- `plan.md`: written by `li-plan`.
- `log.md`: appended by `li-post`/`li-publish` (PostZen post id + URL); read by `li-audit` to match hook formulas to results.

**Data files the skills depend on** (the README and plugin descriptions quote their counts, so keep those in sync when editing):
- `skills/li-post/hooks.json`: 21 hook formulas (`template`, `example`, `best_for`, `trap`).
- `skills/li-human/slop.json`: shared lexicon for both scripts: `invisible` (17 classes), `typographic`, `words` + `phrases` (113 replacements total), `structures` (11 tells). Structural tells are only reported, never auto-rewritten.
- `skills/li-profile/rubric.json`: 12 items summing to 100.

**Plugin wiring:** `.claude-plugin/plugin.json` (version), `.claude-plugin/marketplace.json`, and `.mcp.json` (registers `postzen` at `https://mcp.postzen.dev/mcp`). `.github/workflows/` holds the standard Claude Code `@claude` and PR-review Actions.

## Conventions that matter

- Skills must never fabricate metrics, clients or outcomes; missing facts go in as `{{your number}}` placeholders with a flag.
- No browser automation or scraping of LinkedIn: posting goes only through PostZen or the user's own hands.
- Every writing skill must still work without PostZen connected (the user pastes the result).
- Prose in this repo avoids the things the humanizer strips (em dashes, the slop lexicon), matching the house style of the README and skills.
