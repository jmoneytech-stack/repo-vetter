# repo-vetter

A Claude Code skill that delivers a brutally honest audit of a GitHub repository: what it is, what it does, its pros and cons, its security posture, who it is for, and whether you should install it.

The skill is opinionated by design.
It translates marketing copy into plain category labels, flags non-permissive licenses, and folds security findings into the final install decision.
It never clones or runs the target repository - it audits from metadata, README, structure, and raw file reads only.

## What it does

Given a GitHub URL or `owner/repo`, repo-vetter pulls metadata, README, activity signals, and security signals, then produces a structured, no-fluff verdict.

Notable behavior:

- Fastpath for archived, abandoned, or joke repos - a short verdict instead of the full template.
- Mandatory license callout for anything non-permissive (AGPL, SSPL, BSL, Commons Clause, no-license, and similar).
- A dedicated security audit, with an extra pass for agent tooling (MCP servers, Claude Code skills, browser extensions, editor rules) that gets tool-call privileges inside an LLM session.
- An alternatives section, so a "no" verdict still points you somewhere useful.
- Persists each verdict to memory so repeat audits can be served from cache.

## Install

Clone the skill directly into your Claude Code skills directory:

```bash
git clone https://github.com/jmoneytech-stack/repo-vetter.git ~/.claude/skills/repo-vetter
```

## Usage

Invoke it explicitly:

```
/repo-vetter https://github.com/owner/repo
```

Or just ask in natural language, for example "vet this repo: owner/repo" or "should I install owner/repo?".

Pass multiple URLs to get a head-to-head comparison with a single winner.

## License

MIT - see [LICENSE](LICENSE).
