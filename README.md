Language: English | [日本語](README.ja.md)

# search-first

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/search-first)

An [Agent Skill](https://agentskills.io/specification) for Claude Code that makes the agent look outside before it builds. When you choose a library, look for a prior implementation, check what a paper or an official spec actually says, or are about to write custom code, the agent searches registries, code, papers and official docs, then writes a dated, sourced report. You (or the session that asked) pick from it: adopt a candidate (`Adopt`), adopt one piece of it (`Adopt-part`), or build what no candidate covers (`Build`).

The skill runs web searches and fetches pages through your agent's own web tools, so your search terms leave the machine; it needs no API key of its own. The author's other work is listed under [More from the author](#more-from-the-author).

## Install

As a single skill:

```bash
git clone https://github.com/shimo4228/search-first.git
mkdir -p ~/.claude/skills
cp -r search-first/skills/search-first ~/.claude/skills/
```

Then ask in plain words, such as "is there a library for this?", or type `/search-first`. That is enough for Quick mode, where the agent searches inline for one obvious need; Full mode, for larger questions, also needs a subagent this repository does not ship (see [How It Works](#how-it-works)).

The same skill also ships in the [akc-cycle](https://github.com/shimo4228/akc-cycle) Claude Code plugin, together with the other skills of the Agent Knowledge Cycle (AKC), the author's six-phase loop that turns a coding agent's repeated experience into skills and rules; there it is called `/akc-cycle:search-first`. This repository is synced one way from the same source, so between syncs it can trail the plugin. Take the single skill when you want only this step; take the plugin when you want the whole cycle or the newest copy.

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## Example

An illustrative report (format only; verify live before relying on any item):

```
## Scope searched
"markdown dead link checker" / "link check CI" — npm, crates.io, GitHub code search — 2026-09-14.
Not located: a checker that resolves wiki-style [[links]] in Obsidian vaults.
## Found
1. lychee (github.com/lycheeverse/lychee) — Rust CLI, checks http and local file links,
   `--offline` mode; last release 2026-08 (read 2026-09-14: README + docs/usage.md). Assumes plain
   markdown links; our vault uses [[wikilinks]], so the http check transfers, the local
   resolution does not.
2. textlint-rule-no-dead-link (github.com/textlint-rule/textlint-rule-no-dead-link) —
   runs inside textlint, http only (saw 2026-09-14: README). Transfers only if textlint is already in
   the pipeline; it is not in ours.
## Verdict
http links: Adopt-part lychee: the http check.
[[wikilinks]]: Build — a resolver for wiki-style links; no candidate located covers it.
## Still unknown
Whether either tool can be pointed at a custom link resolver.
```

## How It Works

**Look outside, report, let the caller pick.** The searcher reports what exists outside and how it differs from your situation; the caller (you, the main session, or a session writing a plan) reads the report and decides. The verdict the report gives per need is the searcher's recommendation, which the caller accepts or overrides; in Quick mode (below) one agent is both, and you can override it in chat. The caller adopts an established candidate that matches the need by default and builds only when the report names the gap no candidate fills.

1. **State the question in text.** Before the first tool call, the agent writes 2–3 sentences in chat: the question, its type, and the constraints (language, existing deps, license, budget). This is what lets you redirect it before effort is spent.
2. **Check this repo first**, with one `rg` / Glob pass. The thing may already exist here.
3. **Search by question type.** Each type has its own places to look and its own point to stop:

   | Question type | Where it looks | Stops when |
   |---|---|---|
   | Library / tool choice | registries (npm / PyPI / crates), official docs, release pages | 2–3 candidates have primary facts |
   | Prior implementation | GitHub code search, skill / agent / MCP catalogs, this repo | 2 actual files read, or 3 independent sources agree |
   | Claim of a paper / post | the primary text (arXiv, official blog, the repo file itself) | subject and premises are written down |
   | Spec / official behavior | official docs, `--help`, changelog, running it | one version-stamped primary fact |
   | State of practice | the rows above mixed, plus first-person operating notes | the next search would settle nothing you can name |
   | Primary-source check | the full text the claim points at, past the abstract | every claim is marked supported, partial, unsupported or unverified |

4. **Report.** Every claim carries an as-of date and a source URL, and says whether the full text was read or only a snippet seen (`read` / `saw` with the date in the example above). When nothing turned up, the report writes the range searched, so the next session knows where to widen. Judgement is prose backed by facts: **no numeric scores or grades**. A Verdict section, placed before the closing list of what is still unknown, gives one verdict per need: `Adopt <candidate>`, `Adopt-part <candidate: the piece>`, or `Build` with the gap no candidate fills. When the research is for a plan, the research report leaves the Verdict out and the plan itself carries it.

For a single obvious need the agent runs these steps inline (Quick). For a non-trivial question, or any question asked for a plan, it splits the question into 3–5 angles, one of them adversarial, and spawns one `researcher` subagent (a separate Claude Code agent that searches one angle and writes notes) per angle (Full), each with a budget of 10–15 searches and fetches. Neither this repository nor the akc-cycle plugin ships the `researcher` subagent; to use the author's, copy [`agents/researcher.md`](https://github.com/shimo4228/claude-harness/blob/main/agents/researcher.md) (written in Japanese) from claude-harness into `~/.claude/agents/`. The author's harness also limits where that agent writes with a hook (`research-gate.sh`), which copying the file does not bring.

When you say "skip research", the agent records the skip in one line and proceeds. That line offers a quick scan, which you can still ask for before it builds.

## When It Triggers

- Choosing a library or tool, or adding a dependency
- Looking for a prior implementation before writing a utility, helper or adapter
- Checking whether a paper or post applies here, or checking a claim against its source
- Reading what an official spec does now, or surveying current practice

## More from the author

- **[Where to Put a Coding Agent's Knowledge — and How to Make It Stick](https://dev.to/shimo4228/where-to-put-a-coding-agents-knowledge-and-how-to-make-it-stick-161g)** ([日本語](https://zenn.dev/shimo4228/articles/coding-agent-memory-architecture)): measuring an earlier version of this skill showed that even when the prompt supported searching, the agent skipped comparing candidates and declaring a decision, which is why the report now carries a verdict per need.
- **[Not Reasoning, Not Tools — What If the Essence of AI Agents Is Memory?](https://dev.to/shimo4228/not-reasoning-not-tools-what-if-the-essence-of-ai-agents-is-memory-4k4n)** ([日本語](https://zenn.dev/shimo4228/articles/agent-essence-is-memory)): how search-first, learn-eval and [rules-distill](https://github.com/shimo4228/rules-distill) (which turns principles repeated across skills into rules) grew, skill by skill, into one loop over an agent's memory.
- **[akc-cycle](https://github.com/shimo4228/akc-cycle)**: installs this skill together with the rest of the cycle as one Claude Code plugin.
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: the reasoning behind each phase of the cycle, Research among them, recorded as dated design decisions.
- **[learn-eval](https://github.com/shimo4228/learn-eval)**: the next phase, which keeps what a session taught by saving it where a future session will find it.
- **[signal-first-research](https://github.com/shimo4228/signal-first-research)**: a frozen design note on research intake filters that admit only information likely to change your next action; its principle now lives in skills such as this one.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with AKC next to the other long-running practice lines and their DOIs.

## License

MIT

<details>
<summary>For tools and AI assistants</summary>

search-first is an Agent Skill (one Markdown instruction file in the open Agent Skills format) that makes a coding agent search outside sources at decision time and report what exists, for developers who want the agent to stop reinventing existing libraries and to show where each claim comes from. The searcher reports and recommends one verdict per need; the caller (the user, the main loop, or a planning session) decides what to adopt.

It exists because an agent that builds from memory reinvents what already exists and trusts package facts that have aged. The skill adopts an established candidate that matches the need by default and chooses to build only when the report names the gap no candidate fills. An earlier version, measured with the author's skill-comply in 2026-03 across three prompt scenarios, was followed on searching only when the prompt supported the skill, and even then the agent skipped comparing candidates and declaring a decision (40% of steps followed in that scenario; the numbers are in the article "Where to Put a Coding Agent's Knowledge" linked from the README). The current report shape, with a Verdict section per need, puts that comparison in the report.

Canonical facts: MIT license; a single `SKILL.md` with no code; maintained by one author (@shimo4228). Status: active, synced one way from the author's Claude Code harness by `scripts/sync-from-local.sh` (it never commits), and also shipped in the akc-cycle plugin, so this repository can trail the plugin between syncs. Requirements: an agent with web search and fetch tools; no paid key of its own. Full mode expects a `researcher` subagent that this repository does not ship and writes notes under `$HOME/.cache/claude-research-notes/`; for a plan it writes a report to `<repo>/docs/plans/research/`. An Adopt or Adopt-part that adds a dependency is meant to pass the caller's own dependency-intake step before it lands.

Example: a report in chat has four sections (a report written for a plan leaves out `Verdict`, which the plan itself carries), `Scope searched` (terms, sources, as-of date, and what was not located), `Found` (per item: URL, subject and premises, kind of evidence, how our situation differs, the part that transfers), `Verdict` (per need: `Adopt <candidate>`, `Adopt-part <candidate: the piece>`, or `Build` with the gap), and `Still unknown`. Judgement is prose with dates and versions, never a numeric score.

Links: [skills/search-first/SKILL.md](skills/search-first/SKILL.md) is the skill itself; [llms.txt](llms.txt) and [llms-full.txt](llms-full.txt) are the machine-readable summary and reference. The skill implements the Research phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle), concept DOI [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726); cite AKC by that DOI. The installable form of the whole cycle is [akc-cycle](https://github.com/shimo4228/akc-cycle).

</details>
