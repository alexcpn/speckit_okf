# Changelog

## Moved to catalogify — 2026-09-22

This repository is archived. The Spec Kit extension is now built from
[catalogify](https://github.com/alexcpn/catalogify) and released with it,
starting at [v0.9.2](https://github.com/alexcpn/catalogify/releases/tag/v0.9.2).
Compared with 0.6.0 it adds catalogify's shallow-clone warning and the
narrower V4 check, runs the shell scripts through `bash` (a zip install drops
the executable bit), and gives each command a description in the manifest.
Further changes are recorded in catalogify's releases.

## [0.6.0](https://github.com/alexcpn/speckit_okf/releases/tag/v0.6.0) — 2026-09-07

Ported from the companion [catalogify](https://github.com/alexcpn/catalogify)
project, where every one of these was found by running the workflow against
public repositories and checking what came out.

* **New: `/speckit.okf.verify` — a check on truth, not structure.**
  `validate` only ever asked whether a bundle is well-formed. It said nothing
  about whether any of it is *true*. `verify_okf.py` resolves each claim back
  to the repository: every cited commit must exist and touch that concept's own
  `source_files` (V1, V2) and must not be test-only (V3); every symbol in
  `# Interfaces` must appear in non-test code (V4); a `# Gotchas` section must
  cite something (V5); every `source_files` path must be tracked by git (V8).
  Notes cover dependency links no import backs (V6) and Go "import cycles",
  which the compiler forbids (V7). Run against a kubelet bundle it reproduces,
  unaided, every error previously found by hand — including three test-only
  commits written up as production invariants.

* **`okf-history.sh` lists the files each flagged commit touched, and marks
  test-only ones.** The old output was subject lines, and agents wrote
  invariants from them: "Fix goroutine leak in operation_executor_test.go"
  became "goroutine lifetime is a design property of the plugin manager". The
  instruction not to stop at the subject line already existed, and was ignored.
  A file list in the input cannot be ignored. `[TEST-ONLY]` marks a commit
  whose every file is a test file.

* **New: `okf-cochange.py` — logical coupling from history.** Directories that
  keep changing in the same commit are coupled even when neither imports the
  other, because the mechanism is a wire contract, a shared schema or a
  deployment ordering rule. Reports support, confidence and lift, skipping bulk
  commits so a mass rename does not couple everything to everything.

* **Untracked code is no longer documented as yours.** A run on a real
  repository produced eight concepts describing 952 files of imported
  third-party code sitting in the working tree. The inventory now reports
  `untracked_dirs`, the generate workflow treats them as off-limits, and
  `verify` rejects any concept whose `source_files` git does not track.

* **Cross-links are relative now, and W9 catches the old form.** OKF resolves a
  leading `/` against the *bundle* root; GitHub resolves it against the
  *repository* root. Every cross-link in a published catalog 404'd in a browser
  while validating perfectly. OKF §6.1 permits relative paths, which resolve
  identically for the validator, for an agent, and for a reader clicking
  through.

* **Bundles get a `README.md` front door.** Forges render `README.md` when a
  directory is opened and ignore `index.md`, so a bundle showed a bare file
  list to the humans it was written for. `README.md` is now ignored rather than
  checked as a concept, and the generate workflow specifies writing one. That
  is also where `/speckit.okf.clarify` is now put in front of readers, since
  answering the open questions is the only part of a catalog a machine cannot
  produce.

## [0.5.0](https://github.com/alexcpn/speckit_okf/releases/tag/v0.5.0) — 2026-09-03
* **Fixed: `okf-inventory.sh` aborted with exit 141 on large repositories.**
  Twelve pipelines ended in `head -N`, which closes the pipe and sends
  SIGPIPE upstream; under `set -o pipefail` that became exit 141 and `set -e`
  killed the script. It never fired on small repos, because the producer
  finishes writing before `head` leaves — so it only broke on exactly the
  repositories the tool exists for. Reproduced on `kubernetes/kubernetes`
  (500,022 LOC, 25,917 files, 140,761 commits): every run failed with no
  output. Replaced with a `take()` helper built on awk, which drains to EOF.
  After the fix: exit 0 in 2.11s, writing a 56 KB inventory.
* **Richer, self-verifying concept generation.** The generate workflow was
  producing readable but thin concepts — prose a reader could not check
  against the code, in a bundle an agent could not traverse. Four changes:
  - `# Interfaces` is now **required** for Service and Module concepts and
    must be *extracted*, with per-language grep recipes for Go, Python,
    TS/JS and Java/C#, plus guidance to select ~5–15 caller-relevant entries
    and drop test fakes.
  - `# Dependencies` is now **derived from actual imports**, with extraction
    commands per language, mapping each internal import to the concept whose
    `source_files` owns that path. An orphan concept now explicitly signals
    that this step was skipped.
  - History mining no longer stops at the subject line. The workflow requires
    reading the highest-signal commits (`git show --stat`, full `%B`) and,
    for any `Revert "X"`, finding and reading the original X — the pair is
    what carries the invariant. Adds a "write the rule, not the anecdote"
    instruction with a worked before/after.
  - `open_questions` are now **interrogated** rather than incidental, against
    five fixed categories: guarantees, ordering, failure, compatibility,
    ownership.
* Measured effect of the above on a 9-concept bundle for Kubernetes'
  `pkg/kubelet` (108,648 LOC): bundle +41% (15,835 → 22,386 bytes), open
  questions 2 → 11, cross-links 18 → 27, concepts carrying `# Interfaces`
  2 → 8 — while the **service-level routing entry stayed flat at ~2.7 KB**.
  Added detail lands in module concepts, so it does not cost routing budget.
* **Works on projects that are not under version control.** The inventory
  reported a fabricated `branch: main` outside a repository, and rule E4 made
  a conformant bundle impossible there: every `log.md` date block must carry
  `Commit: <sha>`, and there is no SHA to write. E4 now accepts the literal
  `none` and is skipped entirely when the bundle's repo root has no `.git`;
  the inventory reports a new `git.is_git_repo` boolean and an empty branch,
  so consumers can tell "not a repository" from "repository with no remote".
  The workflows now say what to do in that case: skip history-based reasoning
  rather than inventing it, take `timestamp` from file modification time, omit
  `resource:` without a configured base, write ``Commit: `none` ``, and expect
  to raise more `open_questions` because the "why" can only come from a human.
  `/speckit.okf.update` states up front that it requires git, being a diff
  between two commits.
* `generated_by` bumped to `speckit-okf/0.5.0`.

## [0.4.0](https://github.com/alexcpn/speckit_okf/releases/tag/v0.4.0) — 2026-09-01
* **Packaged as an Agent Skill.** The four workflows are now also available as
  a portable `SKILL.md` skill at
  `plugins/okf/skills/okf-knowledge-bundle/`, usable by Claude Code, Claude.ai,
  the Claude Agent SDK, and any agent that reads the Agent Skills format —
  invoked in plain language instead of by slash command. `SKILL.md` is a small
  router (config/path resolution, script reference, and the rules that hold
  across every workflow); the workflow detail lives in `references/{generate,
  update,clarify,validate}.md` and is loaded only when that workflow runs.
* **Installable as a Claude Code plugin.** Added `.claude-plugin/marketplace.json`
  and `plugins/okf/.claude-plugin/plugin.json`, so
  `/plugin marketplace add alexcpn/speckit_okf` + `/plugin install okf@speckit-okf`
  installs the skill. The plugin deliberately contains only the skill — the
  Spec Kit `commands/` at the repo root hardcode `.specify/` paths and would
  not work outside a Spec Kit project.
* The skill is self-contained: it carries its own copies of the three scripts
  and the config template so the directory works wherever it is copied. The
  repo-root copies stay canonical; **`scripts/sync-skill.sh`** resyncs them and
  `scripts/sync-skill.sh --check` fails CI on drift.
* **New `scripts/python/validate_skill.py`** — lints a skill directory's
  `SKILL.md` frontmatter (name shape and directory match, description length,
  unknown keys, body size) and verifies every skill-relative path it references
  exists. Wired into a new `Skill` workflow alongside the sync and
  manifest-JSON checks; ShellCheck now scans all of `scripts/`.
* The skill also reads `.okf-config.yml` from the repo root, so it can be
  configured without a `.specify/` directory.
* Fixed: the generate workflow's Phase 0 steps were numbered `1,2,5,3,4`, and
  the "bundle already exists" guard ran *after* the inventory scan. The guard
  is now step 2, before any scanning.
* Fixed: every `speckit_ofk` URL (a typo for `speckit_okf`) in the README,
  `SECURITY.md`, `extension.yml`, and CHANGELOG.
* `generated_by` bumped to `speckit-okf/0.4.0`.

## [0.3.0](https://github.com/alexcpn/speckit_okf/releases/tag/v0.3.0) — 2026-07-20
* **Git history as a first-class signal.** `okf-inventory.sh` now emits a
  `git.history` object: `churn` (per-file commit counts over the last
  `OKF_HISTORY_COMMITS` non-merge commits, capped at `OKF_CHURN_TOP`) as a
  significance signal, plus `recent_commits`. Added an `adr_docs` category
  (ADR/RFC/decision files) for seeding Design Decision concepts. All scans
  are bounded and skip cleanly on non-git repos.
* **New `okf-history.sh` script** — bounded, per-path git history for the
  agent to mine the "why" of a concept: creation commit, commit count,
  recent subjects, and revert/hotfix/risk-flagged commits (deadlock, race,
  regression, security). Diff-free by default (`--patch` opt-in) to avoid
  leaking secrets from history; `--json` and `--limit` supported.
* **New `/speckit.okf.clarify` command.** Generate/update now *park*
  uncertainty in an `open_questions` frontmatter list instead of guessing;
  clarify collects those, asks the user in prioritized batches (capped by
  `clarify.max_questions`, default 20), and folds answers back into concept
  bodies marked with `<!-- clarified: ... -->` sentinels. `/speckit.okf.update`
  preserves those sentinels as human curation and never overwrites them.
* `/speckit.okf.generate`: uses churn for Phase 1 significance, runs
  `okf-history.sh` per concept for the "why", and emits `open_questions`
  where code + history are inconclusive. `generated_by` bumped to
  `speckit-okf/0.3.0`.
* `validate_okf.py`: added **W8** (concept has unresolved `open_questions`).
* `validate.md`/`README`/`extension.yml`/config template updated for the new
  command, script, and config knob.

## [0.2.0](https://github.com/alexcpn/speckit_okf/releases/tag/v0.2.0) — 2026-07-17
* `okf-config.yml`'s `exclude` list is now actually honored by both
  `okf-inventory.sh` and `validate_okf.py` (via `--config`/`--exclude`),
  not just interpreted as prompt guidance. Fallback exclude defaults in
  the inventory script's non-git branch now match the config template.
* `validate_okf.py`: added `argparse` (`--config`, `--exclude`,
  `--repo-root`, `--json`, `--help`), graceful error handling instead of
  crashing on unreadable files, and three new checks — **W6** (dangling
  `source_files` entries), **W7** (possible duplicate concept by
  type+title), **E4** (missing/malformed `Commit:` line in `log.md`).
  Extended the W5 secret heuristic to catch unquoted values, AWS-style
  access keys, and PEM private-key blocks. Warns (W0) when PyYAML isn't
  installed and the lenient fallback parser is in use.
* **Breaking (bundle format):** `log.md` date blocks now require a
  `Commit: \`<sha>\`` line as the first line under the heading — this is
  what `/speckit.okf.update` reads to resume incrementally, replacing
  free-form SHA parsing from prose. Bundles generated before 0.2.0 will
  need this line added manually (or regenerated) before `/speckit.okf.update`
  or `/speckit.okf.validate` will treat them as conformant.
* `okf-inventory.sh`: removed dead `json_escape()` helper, applied a
  consistent cap (`OKF_INVENTORY_CAP`, default 150) across all inventory
  categories with a `truncated` flag per category, and switched the
  default output path from the fixed `/tmp/okf-inventory.json` to a
  per-repo, per-PID path to avoid collisions between concurrent runs.
* `/speckit.okf.update`: added an explicit no-op check (stops cleanly if
  `HEAD` already matches the logged commit), explicit handling of renamed
  source files (updates `source_files` in place instead of
  orphaning+duplicating), and a fallback full re-scan when the logged
  commit is no longer reachable (rebase/squash/force-push).
* `/speckit.okf.generate`: the "bundle already exists" guard now checks
  for any `.md` file in `bundle_dir`, not just `log.md`, so hand-seeded
  or partial bundles aren't clobbered.
* `/speckit.okf.validate`: no longer re-derives W4/W5 in prose (relies on
  the validator's own output); the "stale timestamp" spot-check now has a
  concrete algorithm (compare `timestamp` against each `source_files`
  entry's last commit time).

## [0.1.0](https://github.com/alexcpn/speckit_okf/releases/tag/v0.1.0) — 2026-07-17
* Initial release: `/speckit.okf.generate`, `/speckit.okf.update`, `/speckit.okf.validate`.
* Deterministic inventory script and OKF v0.1 conformance validator.
