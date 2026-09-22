# speckit-okf — OKF Knowledge Bundle Generator

[![CodeQL](https://github.com/alexcpn/speckit_okf/actions/workflows/codeql.yml/badge.svg)](https://github.com/alexcpn/speckit_okf/actions/workflows/codeql.yml)
[![ShellCheck](https://github.com/alexcpn/speckit_okf/actions/workflows/shellcheck.yml/badge.svg)](https://github.com/alexcpn/speckit_okf/actions/workflows/shellcheck.yml)
[![Skill](https://github.com/alexcpn/speckit_okf/actions/workflows/skill.yml/badge.svg)](https://github.com/alexcpn/speckit_okf/actions/workflows/skill.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> [!IMPORTANT]
> **This repository has moved into [catalogify](https://github.com/alexcpn/catalogify) and is archived.**
> The Spec Kit extension is now built from catalogify and attached to every
> catalogify release, so it no longer drifts from the CLI and the skill. The
> last release here was 0.6.0.
>
> **Spec Kit users**: reinstall from the catalogify release. The extension ID
> (`okf`), the commands and the config location are unchanged:
>
> ```bash
> specify extension add okf --force --from \
>   https://github.com/alexcpn/catalogify/releases/download/v0.9.2/speckit-okf-0.9.2.zip
> ```
>
> **Agent Skill and Claude plugin users**: switch to catalogify's own skill
> (`uv tool install catalogify && catalogify install`). Remove this repository's
> `okf-knowledge-bundle` skill first, so the two don't compete for the same
> requests.
>
> See [Using it with Spec Kit](https://github.com/alexcpn/catalogify#using-it-with-spec-kit).
> Everything below describes this repository as it was at 0.6.0.

Turns your AI coding agent into an **OKF enrichment agent**: it analyzes a source-code repository and generates a conformant [Open Knowledge Format (OKF v0.1)](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) knowledge bundle — a directory of cross-linked markdown concepts with YAML frontmatter describing your services, modules, APIs, data models, and operations.

Because OKF bundles are plain markdown in git, the generated bundle is readable by humans, diffable in PRs, and consumable by other agents without any bespoke tooling.

Ships in two forms, from one set of prompts and scripts:

| Form | For | Entry point |
| ---- | --- | ----------- |
| **Agent Skill** ([`plugins/okf/skills/okf-knowledge-bundle/`](plugins/okf/skills/okf-knowledge-bundle/SKILL.md)) | Claude Code, Claude.ai, the Claude Agent SDK, and any agent that reads the [Agent Skills](https://code.claude.com/docs/en/skills) `SKILL.md` format | Ask in plain language — "build an OKF knowledge bundle for this repo" |
| **Spec Kit extension** (`commands/`, `extension.yml`) | [Spec Kit](https://github.com/github/spec-kit) users who want explicit slash commands | `/speckit.okf.generate` and friends |

## Workflows

Both forms drive the same four workflows.

| Workflow (skill reference / Spec Kit command) | What it does |
| ------- | ------------ |
| `generate` — `/speckit.okf.generate` | Bootstrap a full bundle from the repo: inventory scan, git-history mining (churn + rationale), concept plan, concept documents, `index.md` files, `log.md`, validation. |
| `update` — `/speckit.okf.update` | Incremental refresh: diffs git history since the last logged commit, surgically updates only stale concepts, deprecates orphans, creates concepts for new code, preserves human curation. |
| `clarify` — `/speckit.okf.clarify` | Resolves the `open_questions` that generate/update parked (instead of guessing) by asking the user, then folds the answers back into concepts as cited, curation-protected knowledge. |
| `validate` — `/speckit.okf.validate` | Runs the OKF §9 conformance checker and a quality spot-check; reports ERRORs/WARNINGs with fixes. Checks **structure**. |
| `verify` — `/speckit.okf.verify` | Checks **claims**. Every cited commit must exist and belong to the concept citing it, every symbol in `# Interfaces` must exist in non-test code, every `source_files` path must be tracked by git, and no invariant may be asserted with no evidence. An agent auditing its own prose is the unreliable case; git is not. |

## Install

### As an Agent Skill

```bash
# Claude Code — as a plugin (adds the skill, keeps it updatable)
/plugin marketplace add alexcpn/speckit_okf
/plugin install okf@speckit-okf

# Any agent that reads SKILL.md — copy the skill directory into place
git clone https://github.com/alexcpn/speckit_okf.git
cp -r speckit_okf/plugins/okf/skills/okf-knowledge-bundle ~/.claude/skills/    # personal
cp -r speckit_okf/plugins/okf/skills/okf-knowledge-bundle .claude/skills/      # this project only
```

The skill is self-contained — `SKILL.md`, the four workflow references, and its
own copies of the scripts — so it works anywhere the directory lands. For the
Claude Agent SDK, point your skill source directory at the same folder.

### As a Spec Kit extension

```bash
# Option 1: install from a released archive (no catalog needed)
specify extension add okf --from https://github.com/alexcpn/speckit_okf/archive/refs/tags/v0.5.0.zip

# Option 2: install from a local clone (dev mode)
git clone https://github.com/alexcpn/speckit_okf.git
specify extension add --dev speckit_okf/

# Option 3: once listed in the community catalog
specify extension add okf
```

## Usage

With the **skill**, just ask — the agent loads it on its own when the request
matches ("generate an OKF knowledge bundle for this repo", "refresh the
knowledge bundle", "validate the OKF bundle").

With the **Spec Kit extension**, run the commands explicitly:

```bash
# 1. Generate the initial knowledge bundle
/speckit.okf.generate

# 2. Resolve anything the agent couldn't infer from code + git history
/speckit.okf.clarify

# 3. After making code changes, refresh incrementally
/speckit.okf.update

# 4. Validate conformance before committing
/speckit.okf.validate

# 5. Check that the bundle's claims survive the repository
/speckit.okf.verify
```

Output lands in `knowledge/` (configurable) as a set of cross-linked markdown concept files ready to commit alongside your code.

## Configuration

Copy `okf-config.template.yml` to `.okf-config.yml` in your repo root (the skill also reads `.specify/extensions/okf/okf-config.yml`, which is where the Spec Kit extension expects it) to control the bundle directory (default `knowledge/`), resource URI base, excludes, type mappings, layout, and granularity (`coarse` / `medium` / `fine`). Defaults work without any config.

## How it works

1. `scripts/bash/okf-inventory.sh` deterministically scans the repo (entry points, dependency manifests, API definitions, migrations/models, CI/CD, docs, ADRs) into JSON — including **git-history signals** (`churn` = per-file commit counts for significance, `recent_commits`) — so the agent plans from facts, not guesses.
2. `scripts/bash/okf-history.sh <path>` gives the agent bounded, per-concept git history — creation commit, recent subjects, and revert/hotfix/risk-flagged commits — so concepts capture the **"why"** (invariants, gotchas) with commit citations, not just the "what".
3. The workflow prompt instructs the agent to draft a concept plan, then write OKF-conformant documents: required `type` frontmatter, recommended `title`/`description`/`resource`/`tags`/`timestamp`, producer extension fields `source_files` (maps each concept back to code — what makes incremental updates possible) and `open_questions` (parks uncertainty for `/speckit.okf.clarify` instead of guessing), relative cross-links that resolve on GitHub, `index.md` progressive-disclosure files, a `README.md` front door, and an ISO-dated `log.md`.
4. `scripts/python/validate_okf.py` enforces OKF §9: parseable frontmatter everywhere, non-empty `type`, reserved-file structure — and warns on broken links, missing indexes, empty bodies, secret-looking strings, dangling `source_files`, duplicate concepts, and unresolved `open_questions`.

## Generated bundle shape (example)

```
knowledge/
├── index.md            # okf_version: "0.1" + directory of everything
├── log.md              # dated history, records source commit SHA
├── architecture/
│   ├── index.md
│   └── overview.md     # type: Reference — the "start here" concept
├── services/…          # type: Service
├── modules/…           # type: Module
├── apis/…              # type: API Endpoint / API Resource
├── data/…              # type: Data Model / Database Table
└── operations/…        # type: Pipeline / Configuration / Playbook
```

## Notes

- The updater never deletes concepts or human-written prose; removed code yields `status: deprecated`, not deletion.
- The agent parks what it can't verify in `open_questions` rather than guessing; `/speckit.okf.clarify` turns those into human-confirmed, curation-protected facts (marked with `<!-- clarified -->` sentinels the updater won't overwrite). **This is the highest-value step, and the only one a machine cannot do for you.**
- History mining marks commits that changed **only test files** as `[TEST-ONLY]` and lists the files each flagged commit touched. A goroutine leak fixed in `foo_test.go` is test hygiene, not a production invariant, and the workflow now says so.
- `okf-cochange.py` mines **logical coupling**: directories that keep changing in the same commit are related even when neither imports the other, because the mechanism is a wire contract, a shared schema or a deployment ordering rule. That relationship exists in no import graph and no snapshot of the working tree.
- The inventory enumerates with `git ls-files`, so `.gitignore` is honoured for free, and it reports untracked subtrees on disk so vendored or imported code is named and skipped rather than written up as if it were yours.
- Secrets found in configs — or surfaced by git-history mining — are described by shape, never by value; the validator flags anything that slips through.
- OKF's permissive consumption model (unknown types OK, broken links OK) is relied on deliberately — generation is safe to run early and often.
- The scripts are checked on every push via CodeQL and ShellCheck; see [SECURITY.md](SECURITY.md) to report a vulnerability.
- Measured on Kubernetes' `pkg/kubelet` (108,648 lines of Go): a 9-concept bundle is 21.9 KB, of which the service-level entry an agent needs for routing is 2.6 KB (~676 tokens). `okf-inventory` scans the full 500k-line Kubernetes tree in 2.1 seconds. On a monorepo, start at `granularity: coarse` and raise `OKF_INVENTORY_CAP` above its default of 150 — and note that raw churn skews toward generated files and build config, so invest in `exclude`.
- The scripts and config template at the repo root are canonical; the skill carries copies so it stays portable. `scripts/sync-skill.sh` resyncs them and CI fails on drift (`scripts/sync-skill.sh --check`).
- `scripts/python/validate_skill.py <skill_dir>` lints the `SKILL.md` frontmatter and its referenced paths; it also runs in CI.

## License

MIT
