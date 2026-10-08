# lens-review

A Claude Code plugin for **multi-agent code review by lenses**. Instead of one reviewer reading the
whole diff, it fans out to specialised subagents, each in its own context and with its own method:

| Lens | Subagent | Looks for |
|---|---|---|
| Correctness | `lens-review:lens-correctness` | logic bugs, edge cases, error handling, concurrency, broken contracts |
| Security | `lens-review:lens-security` | injection, authz/IDOR, SSRF, secrets, insecure defaults, prompt injection |
| Test integrity | `lens-review:lens-tests` | rewritten asserts, tests that do not bite, coverage of the path that can fail |
| Architecture | `lens-review:lens-architecture` | blast radius, coupling, layering, repository conventions (`CLAUDE.md`) |

Every finding is then validated adversarially ("refuted by default") before it reaches the report,
and the report is ranked by severity. **It is advisory only: it never approves or merges.**

The review is tiered by blast radius, not by diff size: a config or docs change gets one reviewer,
a normal feature gets three, and anything touching payments, auth, crypto, migrations,
concurrency or data deletion gets all four plus an intent log.

## Install

```
/plugin marketplace add romerox3/lens-review
/plugin install lens-review@lens-review
```

## Use

```
/lens-review:lens-review            # review the current diff
/lens-review:lens-review 123        # review PR #123 (needs gh)
/lens-review:lens-review 123 --comment   # also post the findings as inline PR comments
```

It also triggers on requests such as "review this PR with several reviewers". Requires `git` and,
for PRs, an authenticated `gh` CLI. The skill's own documentation is in Spanish, with technical
terms in English.

## Using it as a benchmark reference

When comparing this reviewer against others (for example the built-in `/code-review`), record the
plugin version or commit and the Claude Code version with every run. The behaviour depends on both,
and results from different versions are not comparable.

## Layout

```
.claude-plugin/marketplace.json     marketplace catalogue (this repository)
plugins/lens-review/
  .claude-plugin/plugin.json        plugin manifest
  agents/                           the four lens subagents
  skills/lens-review/               the orchestrator skill: prompts, lenses, tiers, templates, evals
```
