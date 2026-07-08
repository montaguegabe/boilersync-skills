---
name: updating-high-prio-projects
description: >-
  Use this skill when Gabe asks to update high-priority downstream BoilerSync projects, apply template changes across prioritized projects, review Gabe Priority BoilerSync reports, or bring prioritized BoilerSync instantiations up to date without blindly overwriting project code.
version: 0.1.0
---

# Updating High-Priority BoilerSync Projects

Run template updates across Gabe-prioritized BoilerSync downstream projects safely, using BoilerSync diffs and pull proposals. This workflow assumes most downstream updates need AI or manual judgment.

## Required Related Skills

Also use the `boilersync` skill whenever this skill is active. Use `git-dirty-commit` if Gabe asks to commit or push the resulting changes.

## Core Principles

- Do not use direct `boilersync pull` as a bulk updater for customized downstream projects.
- Prefer `boilersync pull-proposal create --json`; inspect the proposal and apply only safe files or hunks.
- Treat `.starter` files as project-owned after initialization. Use `--include-starter` only when deliberately adding or reviewing starter-derived files.
- Do not add template-specific detection to BoilerSync. Nested template tracking is explicit via `.boilersync` `children`.
- Default to one catch-all app/package for project-specific code. Split into child BoilerSync instantiations only when Gabe deliberately wants separate tracking.
- Trash, `.local`, and scratch paths are never priority unless Gabe explicitly says otherwise.
- If Gabe marks any instantiation in a project or multi-workspace as `GABE PRIORITY`, treat all relevant BoilerSync instantiations in that project/workspace as high priority.

## High-Priority Update Workflow

1. Establish the priority set.
   - Start from the latest RMOT or priority report if one exists.
   - Expand each priority project/workspace to all relevant `.boilersync` roots under that project.
   - Exclude trash, `.local`, and temporary copies.
   - Record the paths grouped by template, usually lower-level templates first.

2. Check status before changing anything.
   - Run `git status --short --untracked-files=all` for every affected repo root.
   - Keep existing dirty changes. Assume they are user/agent work and do not revert them.
   - If a repo is dirty, direct `boilersync pull` should not be used; proposals are still safe because they work in temp copies.

3. Inspect current downstream drift.
   - Use rendered-template diffs:
     ```bash
     BOILERSYNC_ROOT_DIR=/path/to/instantiation boilersync diff --json
     BOILERSYNC_ROOT_DIR=/path/to/instantiation boilersync diff --json --include-starter
     ```
   - Use `--children` or `--recursive` only for explicitly registered children.
   - Interpret remaining drift as either template-owned change, starter-owned project change, or project-owned extra code.

4. Create pull proposals instead of direct pulls.
   - For each instantiation:
     ```bash
     BOILERSYNC_ROOT_DIR=/path/to/instantiation \
       boilersync pull-proposal create --json \
       --workspace-dir /tmp/boilersync-update-proposals/<template>_<project>
     ```
   - Add `--include-starter` only when intentionally adding/reviewing starter files:
     ```bash
     BOILERSYNC_ROOT_DIR=/path/to/django-app \
       boilersync pull-proposal create --json --include-starter \
       --workspace-dir /tmp/boilersync-update-proposals/django-app_<project>
     ```
   - Inspect with:
     ```bash
     git -C /tmp/boilersync-update-proposals/<template>_<project>/proposal diff
     ```

5. Apply only safe proposal pieces.
   - Whole-file apply is appropriate for clearly safe files such as `.boilersync` metadata or newly added empty/skeleton files:
     ```bash
     BOILERSYNC_ROOT_DIR=/path/to/instantiation \
       boilersync pull-proposal apply-file PROPOSAL_DIR .boilersync
     ```
   - For customized files such as `pyproject.toml`, settings, entry points, app code, serializers, views, and tests, prefer manual or selected-hunk edits.
   - Never apply a whole proposed file if the proposal would delete project dependencies, settings, entry points, or domain logic.

6. Handle common template updates carefully.
   - Hatch package inclusion: add only the missing section/field while preserving project dependencies and existing Hatch config.
   - New starter test layout: add missing `tests/` package files, then move existing `tests.py` into source-named modules when appropriate.
   - BoilerSync metadata: update `.boilersync` with the current template commit when the template update has been applied or intentionally reviewed.
   - Agent guidance cleanup: remove stale `.cursor/rules/*` only when replacing it with the chosen `AGENTS.md`/`CLAUDE.md` convention for that repo.

7. Verify after applying changes.
   - For each instantiation:
     ```bash
     BOILERSYNC_ROOT_DIR=/path/to/instantiation boilersync check-pull
     ```
     The goal is `Due for pull: no` for reviewed high-priority instantiations.
   - Re-run `boilersync diff --json`; remaining drift should be intentional project-owned or starter-owned drift.
   - Validate edited config files with the relevant parser, for example `tomllib` for `pyproject.toml`.
   - Run focused lint/tests for moved files, then broader tests when shared tooling changed.

8. Report clearly.
   - If Gabe asks for a report/RMOT, write Markdown to `/tmp` and open it in Typora.
   - Summarize:
     - which templates and downstreams were updated,
     - which proposal pieces were applied,
     - which proposal changes were intentionally skipped as project-specific,
     - verification results,
     - remaining intentional drift or risks.

## Useful Command Patterns

Generate proposals from a TSV inventory:

```bash
mkdir -p /tmp/boilersync-update-proposals
while IFS=$'\t' read -r template project root extra; do
  workspace="/tmp/boilersync-update-proposals/${template}_${project}"
  BOILERSYNC_ROOT_DIR="$root" \
  BOILERSYNC_TEMPLATE_DIR="${BOILERSYNC_TEMPLATE_DIR:-$HOME/.boilersync/templates}" \
    boilersync pull-proposal create --json --workspace-dir "$workspace" $extra \
    > "/tmp/boilersync-update-proposals/${template}_${project}.json"
done < /tmp/boilersync-update-proposals/paths.tsv
```

Summarize proposal outputs:

```bash
jq -r '"\(.proposal_dir)\t\(.changed_count)\t" + ([.changed_files[].path] | join(","))' \
  /tmp/boilersync-update-proposals/*.json | sort
```

Apply a safe file:

```bash
BOILERSYNC_ROOT_DIR=/path/to/instantiation \
  boilersync pull-proposal apply-file PROPOSAL_DIR .boilersync
```

Apply selected hunks:

```bash
git -C PROPOSAL_DIR diff -- pyproject.toml > /tmp/pyproject-template-update.patch
# edit the patch down to the desired hunks
BOILERSYNC_ROOT_DIR=/path/to/instantiation \
  boilersync pull-proposal apply-patch /tmp/pyproject-template-update.patch
```

Check due status:

```bash
BOILERSYNC_ROOT_DIR=/path/to/instantiation \
  BOILERSYNC_TEMPLATE_DIR=~/.boilersync/templates \
  boilersync check-pull
```

## Warnings And Gotchas

- In zsh, do not name a loop variable `path`; it mutates `PATH`. Use `project_path` or `root`.
- If GitPython cannot find `git` in a non-login shell, set `GIT_PYTHON_GIT_EXECUTABLE="$(command -v git)"`.
- `boilersync diff` may still show cache/build noise if those files exist downstream; treat `__pycache__`, build outputs, and runtime caches as noise unless the template explicitly owns them.
- Proposal directories are disposable. Do not edit them as source of truth; apply or manually port the selected changes into the real project.
- `check-pull` saying `Due for pull: no` means the instantiation metadata has been brought to the current template commit, not that all project drift should disappear.
