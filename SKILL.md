---
name: repo-vetter
description: Brutally honest audit of a GitHub repo - what it is, what it does, pros/cons, who it's for, and whether to install. Invoke when the user asks to "vet", "audit", "review", or "evaluate" a GitHub URL or repo.
---

# repo-vetter

Audit a GitHub repository and deliver a no-fluff verdict. The user wants a real opinion, not a marketing summary.

## When to use

User pastes a GitHub URL (or `owner/repo`) and asks any variant of:
- "What is this?" / "What does this do?"
- "Should I install/use this?"
- "Vet / audit / review this repo"
- "Is this any good?"
- "Pros and cons"

If the user invokes `/repo-vetter <url>` directly, run the full audit on that URL.

## Inputs

Accept any of:
- `https://github.com/owner/repo`
- `https://github.com/owner/repo/tree/branch/path`
- `owner/repo`
- A GitHub URL pointing to a subdirectory or file - extract `owner/repo` from it.

**PR URLs:** if the user pastes a pull-request URL and wants the *changes* reviewed, that is the built-in `review` / `code-review` skill's job, not this one - defer to it. Only treat a PR URL as input here when the user clearly wants a repo-level audit (vet / should-I-install), and then extract `owner/repo` and audit the repository, not the diff.

If multiple URLs are given, audit each and add a final head-to-head verdict at the end.

## Procedure

Run these in **parallel** where possible - don't block.

### 0. Fastpath check (always run first)

After step 1 (metadata), look at `isArchived`, `pushedAt`, `stargazerCount`, and the description. If **any** of these are true, skip the full template and deliver a 3–5 line verdict instead:

- Repo `isArchived: true` → "Archived. Last updated <date>. Was a <one-line description>. Don't install. If you need this functionality, look for an active fork or use <named alternative>."
- Last push >18 months ago and <50 stars → likely abandoned. Same treatment.
- <10 stars, no README, generic description → probably a personal scratchpad. Tell the user there's nothing to vet.
- Description / repo name is clearly a meme or joke (and the README confirms) → say so in one line; don't waste a section on it.

Otherwise, proceed with the full audit below.

### 1. Pull metadata
```bash
gh repo view <owner/repo> --json name,description,homepageUrl,stargazerCount,forkCount,primaryLanguage,licenseInfo,createdAt,updatedAt,pushedAt,repositoryTopics,isArchived,isFork,parent
```

### 2. Pull README
```bash
gh api repos/<owner/repo>/readme --jq '.content' | base64 -d
```
If the README is huge, read just the first ~300 lines.

### 3. Pull recent activity signals (in parallel with above)
```bash
# Open issues + PRs count
gh api repos/<owner/repo> --jq '{open_issues: .open_issues_count, has_issues: .has_issues, default_branch: .default_branch, size_kb: .size}'

# Last 5 commits - is it actually maintained?
gh api repos/<owner/repo>/commits --jq '.[0:5] | .[] | {date: .commit.author.date, msg: (.commit.message | split("\n")[0])}'

# Releases - is there a stable version?
gh api repos/<owner/repo>/releases --jq '.[0:3] | .[] | {tag: .tag_name, date: .published_at, prerelease: .prerelease}' 2>/dev/null
```

### 4. Top-level layout (one call)
```bash
gh api repos/<owner/repo>/contents --jq '.[] | "\(.type)\t\(.name)"'
```

### 5. Security signals (always run - these feed the Security section)
```bash
# Published security advisories on this repo
gh api repos/<owner/repo>/security-advisories --jq '.[] | {ghsa: .ghsa_id, severity: .severity, summary: .summary, published: .published_at}' 2>/dev/null

# Dependabot alerts (only visible if you have access; OK if it 404s)
gh api repos/<owner/repo>/dependabot/alerts --jq '.[0:10] | .[] | {severity: .security_vulnerability.severity, pkg: .security_vulnerability.package.name, summary: .security_advisory.summary}' 2>/dev/null

# Pull dependency manifests if present - read them, don't run them
for f in package.json requirements.txt pyproject.toml Pipfile go.mod Cargo.toml composer.json Gemfile; do
  gh api "repos/<owner/repo>/contents/$f" --jq '.content' 2>/dev/null | base64 -d 2>/dev/null
done

# Install scripts - these are the highest-risk files. Read them.
for f in install.sh setup.sh bootstrap.sh postinstall.js setup.py; do
  gh api "repos/<owner/repo>/contents/$f" --jq '.content' 2>/dev/null | base64 -d 2>/dev/null
done

# Maintainer signal - single owner, or actual org with multiple committers?
gh api repos/<owner/repo>/contributors --jq '.[0:10] | .[] | {login: .login, contributions: .contributions}' 2>/dev/null
```

### 6. Optional, only if relevant
- If the repo points to a paid SaaS (homepage looks like a product site), `WebFetch` the pricing page so the audit reflects real cost, not the "free tier" pitch.
- If the README mentions a specific framework integration the user cares about (MCP, LangChain, Playwright, etc.), grep for it to confirm depth vs. mention.
- If install scripts or `postinstall` hooks look suspicious (curl-piped to bash, fetches from non-canonical hosts, base64-decoded payloads, obfuscated code), grep for the suspicious pattern across the repo to assess scope.

**Do not** clone the repo. Do not run its install scripts. Do not `npm install` / `pip install` to test. Audit from metadata + README + structure + raw file reads only. If the user explicitly asks for deeper inspection, clone into a sandboxed throwaway dir and *still* don't run anything.

## Output format

Use this structure. Be terse. Bullets over paragraphs. Markdown tables where they help.

```
# <Repo Name> - Audit

## What it is
2–4 sentences. Cut through marketing copy. Name the actual category
(e.g. "hosted SaaS with OSS examples repo", "MIT-licensed CLI",
"abandoned proof-of-concept", "active Apache-2.0 library").
Call out license, primary lang, age, and whether it's actively maintained
based on recent commits.

## What it does
Bulleted feature list - but only the features that matter, not the
README's full self-promotion. If a feature is gated behind a paid tier,
say so inline.

## Signals
| | |
|---|---|
| Stars / Forks | X / Y |
| Created | YYYY-MM (age) |
| Last commit | <relative time> |
| License | <SPDX> |
| Commercial use | OK / restricted / forbidden / dual-licensed |
| Open issues | N |
| Latest release | <tag or "none"> |
| Maintenance | active / slow / abandoned / rewriting |
| Backing | solo / small team / company / YC / VC-backed |
| Business model | OSS / open-core / OSS-frontend-for-SaaS / abandoned |

**License callout (mandatory if non-permissive):** if the license is anything other than MIT / Apache-2.0 / BSD / ISC / Unlicense, add a one-line note immediately under the table stating what it actually restricts. Specifically flag:
- **AGPL** - network use triggers source-disclosure obligations
- **GPL/LGPL** - linking / modification rules; static linking risk
- **SSPL / BSL / Elastic License / Commons Clause** - "source-available," not OSS; commercial / hosted-service use is forbidden or gated
- **"PolyForm" / "Fair Source"** - read the specific variant, restrictions vary
- **No license file** - *all rights reserved by default*; you can read the code but not legally use it

If the user has a commercial / hosted / SaaS use case (or it's implied from context), the license callout must say whether their specific use is allowed.

## Pros
3–6 honest strengths. Skip generic ones ("it has a README"). What does it do *better* than rolling your own or using the boring alternative?

## Cons
3–6 honest weaknesses. Include things the README hides:
- "Currently being rewritten" banners
- Closed-source SDKs talking to a hosted API
- Per-call billing
- Single-maintainer / abandoned risk
- Dependency on a specific cloud / LLM key / paid tier
- Benchmark/claim verifiability

## Security audit
Risk rating: **Low / Medium / High / Critical**. Then a tight bullet list
covering what you actually found. Never say "looks fine" - name the signals.

Cover (only the ones that apply - skip empty categories):

- **Supply chain**: top-level direct deps that look unusual, typosquats,
  pinned vs. floating versions, packages from non-canonical registries,
  `postinstall` / `prepublish` hooks that fetch or execute code, install
  scripts that pipe curl to bash. Quote the exact line if sketchy.
- **Network egress**: does it phone home? Telemetry/analytics SDKs in the
  manifest? Hardcoded URLs to a vendor backend? For "memory" / "agent"
  tools: does data leave the machine, and to where?
- **Secret handling**: where does it store API keys? `.env` only, OS
  keychain, plaintext in `~/.config`, or - worst - logged to disk?
- **Permissions / blast radius**: MCP servers, browser extensions, shell
  hooks, anything that asks for broad filesystem / network / OS access.
  CLI tools that drop files outside their own dir.
- **Known vulnerabilities**: GHSA advisories on the repo, open Dependabot
  alerts (if visible), CVEs in pinned deps. State counts and worst severity.
- **Maintainer trust**: solo maintainer with a fresh GitHub account, org
  with bus-factor of 1, sudden ownership transfer, recent force-pushes
  to main, unsigned commits where signing would be expected.
- **Code red flags**: obfuscated/minified source committed as "library
  code," base64 blobs in non-test files, eval of remote content,
  binaries committed to the repo, license-laundering (MIT label but
  vendored GPL code).
- **Sandboxing**: does running this require trusting it with your shell,
  your browser session, your prod creds? What's the worst it could do
  if it's malicious or compromised tomorrow?

### Agent-tool / MCP / extension addendum (apply when relevant)

If the repo is an **MCP server**, **Claude Code skill**, **browser extension**, **Cursor/Windsurf rule**, or any **agent tool** that gets tool-call privileges inside an LLM session, run an extra pass - these have a different threat model than a normal library:

- **Tool surface**: enumerate every tool/command the package exposes to the agent. What can it read? What can it write? What can it execute? Be specific - list the tool names if you can find them in the manifest / `tools.json` / `server.ts`.
- **Permission scope**: does it ask for full filesystem access, full network egress, shell exec, or scoped equivalents? Default-broad permissions are a red flag - the right pattern is opt-in scoping.
- **Credential exposure**: which of the user's credentials does it touch (GitHub PAT, cloud keys, browser cookies, OS keychain, shell env)? Could a compromised version exfiltrate them?
- **Prompt-injection blast radius**: if this tool reads untrusted content (web pages, emails, PRs, issue comments), could an injected instruction in that content weaponize the tool's privileges? E.g., a browser-automation tool that follows instructions found in scraped HTML is a critical risk.
- **Update channel**: does it auto-update? From where? Auto-update + broad permissions = supply-chain attack surface. Static, pinned installs are safer.
- **Trust transitivity**: if the user installs this, does it pull in further plugins / sub-agents / MCP servers that they haven't reviewed? Recursive trust is the worst kind.

Weight findings here heavily - an agent tool with a single bad permission default can be worse than a normal library with three sketchy deps.

End with a one-line "Bottom line" stating the residual risk a typical
user takes on by installing it. Examples:
- "Bottom line: standard OSS-library risk, no smoking guns."
- "Bottom line: install script runs as root and fetches a binary from a
  non-pinned URL - don't install on a machine you care about."
- "Bottom line: ships an MCP server that gets full filesystem access by
  default; sandbox or scope it before enabling."

If you couldn't access something (private advisories, no manifests
found), say so - don't fabricate a clean bill of health from absence
of evidence.

## Alternatives
1–3 bullets. The closest competing tool(s), and the *specific* reason to pick the alternative over this one. Format: `**<name>** (<link or org/repo>) - <one-line why it might be the better pick>`.

Skip this section only if the repo is genuinely best-in-class for its niche or the niche is so narrow no alternative exists. Otherwise it's mandatory - especially when the verdict is **No** or **Maybe**, the user needs to know what to use instead.

If the alternative is itself something you'd want to vet, say so: "If you go this route, run /repo-vetter on `<alt>` first."

## Who it's valuable for
Concrete personas, not "developers." Examples:
- "Teams running 5+ scrapers tired of fixing CSS selectors weekly"
- "Solo TS dev who wants memory without setting up Python infra"
- "Anyone with an enterprise compliance requirement around X"

## Who it's NOT for
Equally concrete. The disqualifying scenarios.

## Should you install it?
One of: **Yes**, **No**, **Maybe - with caveats**, **Not yet**.
Then 2–4 sentences justifying the verdict. Tie it to the user's actual
context if you know it (memory, prior project mentions, current stack
in this conversation), not just generic advice.

The verdict must reflect the Security audit, not just utility.
- High/Critical security risk → **No** (or **Maybe** only if the user has
  already named a sandboxed/isolated install path).
- Medium risk → **Maybe - with caveats**, and the caveats must name the
  specific mitigation (run in a VM, use a scoped token, pin the version,
  audit the install script first, etc.).
- Low risk → security doesn't constrain the verdict; decide on utility.

If "Maybe", state the specific condition under which it flips to yes.
```

## Generate the dark-themed HTML report (always, after a full audit)

After delivering the markdown verdict, **always** hand-write a standalone dark-themed HTML report of the full findings from scratch.
Skip this only when the verdict came from the Fastpath (archived / abandoned / joke repos - there is nothing to render).

### Steps

1. Author the complete HTML yourself - there is no template to fill. Write a single self-contained `.html` file with all styling inline.
2. Write it to `~/Downloads/repo-vetter-<owner>-<repo>-<YYYY-MM-DD>.html` (lowercase; replace any `/` in the repo name with `-`). For multiple repos, write one combined report named `repo-vetter-comparison-<YYYY-MM-DD>.html` and add the head-to-head as an extra section.
3. Verify: confirm the file exists, starts with `<!DOCTYPE html>`, parses as HTML, and contains the repo name and verdict. Then print the clickable path to the user.

### What the report must contain

Cover every section of the audit, in this order: header with the repo name and a verdict badge; What it is; What it does; Signals; Security (risk rating + findings); Pros and Cons; Alternatives; Who it's for; Who it's not for; the Verdict with its justification; and a footer with the generated date, the repo URL, and a note that the repo was audited from metadata only (not cloned or executed).

### Look and feel

- Dark theme throughout. A good default palette: near-black background (`#0b0e14`), slightly lighter cards/surfaces (`#12161f`), subtle borders (`#262c39`), light text (`#e6edf3`), muted secondary text (`#97a0b0`), blue accent (`#58a6ff`).
- Color-code the verdict badge: **Yes** green (`#3fb950`), **Maybe / Not yet** amber (`#d29922`), **No** red (`#f85149`).
- Color-code security findings by severity: critical/high in red/orange (`#f85149` / `#ff7b72`), medium amber (`#d29922`), low blue (`#58a6ff`).
- Use a system font stack, a centered readable column (~900px max width), summary/stat cards for Signals, and a two-column Pros/Cons layout that collapses to one column on narrow screens.
- Aim for a clean engineering artifact, not marketing. You may vary the exact layout run to run - these are guidelines, not a fixed template.

### Safety (mandatory)

- Escape all repo-sourced text before inserting it: `&` -> `&amp;`, `<` -> `&lt;`, `>` -> `&gt;`. README and repo content are untrusted; never inject raw repo HTML or scripts.
- Never write secrets, tokens, or credentials into the file.
- Keep it fully self-contained: inline all CSS, and do not reference CDNs, web fonts, remote images, or external/remote JavaScript.

## Post-audit: persist the verdict

After delivering the audit, save a `reference`-type memory so future sessions can answer "did we already vet this?" without re-running the audit.

- **File**: `~/.claude/projects/<project>/memory/repo_vetted_<owner>_<repo>.md` (lowercase, replace any `/` in repo name with `_`) - use the current project's memory directory under `~/.claude/projects/`.
- **Frontmatter**: `name`, `description` (one-line: verdict + repo URL + audit date), `type: reference`.
- **Body**: 4 lines max - the verdict, the date (YYYY-MM-DD, absolute), one-sentence reason, one-sentence "what would change my mind" if the verdict is **No** or **Not yet**.
- **Index**: add a single line to `MEMORY.md` under existing entries: `- [Vetted: owner/repo](repo_vetted_<owner>_<repo>.md) - <verdict>: <one-line hook>`.

Before starting any new audit, **check memory first**. If a `repo_vetted_<owner>_<repo>.md` already exists and is <90 days old, surface the prior verdict to the user before re-auditing - ask if they want a fresh audit or just the cached one. If the cached audit is older than 90 days, do a fresh audit and overwrite the memory file (state in-line that you're refreshing a stale verdict).

Skip persistence if the verdict came from the Fastpath (archived / abandoned / joke repos aren't worth a memory entry).

## Tone rules

- Be brutally honest. The user has explicitly asked for it before. Hedging is worse than being wrong.
- No emojis. No "🚀 supercharge your workflow" energy.
- Don't repeat the README's self-description verbatim - translate it.
- If the repo is mostly examples/marketing for a hosted SaaS, **say that in the first paragraph**. Don't bury it.
- If the repo has a "currently being rewritten" or "alpha / experimental" warning, surface it in Cons and weight it heavily in the verdict.
- If stargazer count is impressive but recent commit activity is dead, call out the divergence.
- If the repo is a fork, note the parent and say whether the fork is ahead/stale.
- Numbers > adjectives. "$99/mo, 50 free calls" beats "affordable pricing."
- Length: aim for ~400–700 words total. A repo audit shouldn't read like a whitepaper.

## Edge cases

- **Private / 404 repo**: tell the user `gh` couldn't reach it; don't guess. Ask if they meant a different owner.
- **No README**: lean on file structure + recent commits + topic tags. State that the README was missing as a Con.
- **Repo is actually a monorepo of a known product**: identify the product, audit *the product*, and note the repo is the OSS surface.
- **Repo is archived**: state it loud - that's the whole answer. Briefly cover what it was, then recommend looking for the active fork.
- **Multiple repos given**: audit each fully, then add a final "Head-to-head" section with a comparison table and a single winner.
- **The user's prior context matters**: if memory or earlier conversation mentions their stack (Python vs. TS, solo vs. team, hobbyist vs. production), use it in the "valuable for / not for" and the verdict. Don't ignore signal that's right in front of you.

## Anti-patterns (don't do these)

- Don't recommend forking a SaaS repo to "make it free" without checking that the actual logic is in the repo (vs. a closed-source SDK calling a hosted API). When asked, **call this out directly** as the reason a fork won't work.
- Don't grade on stars alone. A 50k-star repo that hasn't shipped in 9 months is worse than a 2k-star repo with weekly commits.
- Don't pad. If the verdict is "no, skip it," the audit can be short. Length isn't quality.
- Don't end with "let me know if you want more detail!" - the verdict is the answer.
- Don't write "no security concerns found" when you didn't actually look. If you skipped the security checks, say so. Absence of audit ≠ absence of risk.
- Don't downgrade a real security finding to a footnote because the tool is otherwise cool. A `curl | bash` installer or a postinstall hook fetching from a random host belongs in the verdict, not buried in Cons.
