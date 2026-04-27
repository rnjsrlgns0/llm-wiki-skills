---
name: project-lint
description: Use when the user says "프로젝트 위키 점검", "project lint", "위키 건강검진", or after major changes. Runs the project wiki lint checklist — detects orphan pages, contradictions between ADRs and code, stale module docs, missing cross-refs, index drift — and records findings in log.md.
---

# Skill: project-lint

> Core principle: "결정의 맥락이 코드보다 오래 산다"
> (The context behind a decision outlives the code it produced.)

---

## Overview

This skill audits the health of a project wiki. It runs 8 structured checks against the wiki's pages, frontmatter, cross-references, and directory structure. Results are recorded in `log.md` and a report is presented to the user. Auto-fixable issues are resolved immediately and reported. Issues requiring judgment are surfaced for user decision.

---

## Reference: Page Types

The wiki recognizes 6 canonical page types plus 1 inbox type:

| Type        | Purpose                                        |
|-------------|------------------------------------------------|
| `adr`       | Architecture Decision Record                   |
| `module`    | Source module / component documentation        |
| `glossary`  | Term definitions                               |
| `runbook`   | Operational procedures                         |
| `incident`  | Incident post-mortems                          |
| `changelog` | Release or change history                      |
| _(inbox)_   | `_inbox/` draft pages pending classification  |

---

## Reference: Frontmatter Required Fields

Every page (except `_inbox` drafts) must contain all of the following frontmatter fields:

```yaml
---
id: <matches-filename-slug>
title: <human-readable title>
project: <project-name>
type: <one of: adr | module | glossary | runbook | incident | changelog>
status: <valid status for the type — see below>
tags: [...]
refs: [...]
related: [...]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

**Valid `status` values by type:**

| Type        | Valid statuses                                      |
|-------------|-----------------------------------------------------|
| `adr`       | `proposed`, `accepted`, `superseded`, `deprecated`  |
| `module`    | `active`, `outdated`, `deprecated`                  |
| `glossary`  | `active`, `deprecated`                              |
| `runbook`   | `active`, `outdated`, `deprecated`                  |
| `incident`  | `open`, `resolved`                                  |
| `changelog` | `active`                                            |

---

## Reference: Expected Directory Structure

```
project-wiki/<project-name>/
├── index.md          # master index of all pages
├── log.md            # append-only audit/activity log
├── adrs/             # ADR pages
├── modules/          # module documentation pages
├── glossary/         # glossary term pages
├── runbooks/         # runbook pages
├── incidents/        # incident pages
├── changelog/        # changelog pages
└── _inbox/           # draft / unclassified pages
```

---

## Execution Protocol

### Step 0 — Confirm target project

Before running any check, confirm the target project with the user if it has not been explicitly stated. Do not assume or infer the project from context.

- Ask: "어느 프로젝트의 위키를 점검할까요? (project-wiki/ 하위 디렉토리 이름을 알려주세요)"
- Proceed only after receiving explicit confirmation.
- One project at a time — do not run across multiple projects in a single session.

---

### Check 1 — Orphan Pages

**Definition**: A page is an orphan if no other page references it via either a `[[wikilink]]` in body text or a `related` frontmatter entry.

**Procedure**:
1. Collect all page slugs in the project directory (excluding `index.md` and `log.md`).
2. For each slug, scan all other pages' body text for `[[slug]]` patterns and all `related` arrays for the slug.
3. Pages with zero inbound references are orphans.

**Action**:
- List each orphan with its type and `updated` date.
- Do NOT delete or archive automatically.
- Suggest linking (if related content exists) or archiving (if stale). User must approve.

---

### Check 2 — ADR-Code Contradictions

**Definition**: An ADR documents a decision. A contradiction exists when:
- An `accepted` ADR asserts "use X" but the codebase uses Y instead.
- An `accepted` ADR references files, functions, libraries, or patterns that no longer exist or have been replaced.

**Procedure**:
1. Collect all ADRs with `status: accepted`.
2. For each ADR, identify the technical claims (libraries, patterns, file paths, function names) mentioned in the body.
3. Cross-check those claims against the current codebase (file existence, import statements, dependency files).
4. Flag mismatches where code diverges from the ADR's stated decision.

**Action**:
- Report each contradiction as: ADR slug + the specific claim + what the code shows instead.
- Do NOT change ADR status automatically.
- Present to user for decision: update ADR status to `superseded` or `deprecated`, or update the code to realign.

---

### Check 3 — Stale Module Docs

**Definition**: A module page is stale when:
- Its `updated` frontmatter date is older than the last-modified date of the source files it documents.
- It references file paths, function names, or class names that no longer exist in the codebase.

**Procedure**:
1. Collect all pages with `type: module`.
2. For each module page, extract file path references from the body.
3. Compare the page's `updated` date against the filesystem `mtime` of those files.
4. Check that referenced symbols (files, functions, classes) still exist.

**Action**:
- Auto-fix: set `status: outdated` in the frontmatter if the page is detectably stale. Report the change.
- Flag for user: list broken references (files/functions that no longer exist) with suggested refresh actions.

---

### Check 4 — Missing Cross-References

**Definition**: Implicit relationships exist in the content but are not expressed as wikilinks.

**Specific patterns to detect**:
- An ADR body mentions a module name that exists as a page, but no `[[module-slug]]` link is present.
- An incident page mentions a runbook by name but has no `[[runbook-slug]]` link.
- A changelog entry references a PR or feature that corresponds to an ADR, but no `[[adr-slug]]` link is present.

**Procedure**:
1. For each page, extract proper nouns, module names, and runbook names from body text.
2. Attempt to resolve each against known page slugs.
3. Flag resolved matches that lack a wikilink.

**Action**:
- Auto-fix: insert `[[slug]]` wikilinks for unambiguous matches (exact slug match). Report all insertions.
- Flag for user: cases where the match is ambiguous (multiple candidate slugs) — present options and ask which to link.

---

### Check 5 — Superseded ADR Chain Integrity

**Definition**: Every ADR with `status: superseded` must reference the ADR that supersedes it. A "dangling" superseded ADR has no such reference.

**Procedure**:
1. Collect all ADRs with `status: superseded`.
2. For each, check the body and `related` field for a reference to a superseding ADR (`[[adr-slug]]` where the target has `status: accepted` or `status: superseded` itself).
3. Flag any superseded ADR that has no superseding reference.

**Action**:
- Do NOT auto-insert the superseding reference (the correct superseding ADR must be determined by the user).
- Report each dangling superseded ADR with a prompt: "Which ADR superseded this one?"

---

### Check 6 — Index Consistency

**Definition**: `index.md` must be a complete and accurate registry of all pages in the project wiki.

**Procedure**:
1. List all `.md` files in the project directory tree (excluding `index.md`, `log.md`, and `_inbox/`).
2. Parse `index.md` for page entries (wikilinks or listed slugs).
3. Identify:
   - Files present in the filesystem but missing from `index.md` (new/unregistered pages).
   - Entries in `index.md` that point to files that do not exist (stale entries).

**Action**:
- Auto-fix: add missing files to `index.md` with their `title` and `type` from frontmatter. Report all additions.
- Auto-fix: remove `index.md` entries pointing to deleted files. Report all removals.
- Both auto-fixes must be reported with explicit before/after change lists.

---

### Check 7 — _inbox Backlog

**Definition**: Files accumulating in `_inbox/` without being classified represent a documentation debt.

**Procedure**:
1. Count all `.md` files in `_inbox/`.
2. If count > 5: trigger recommendation.

**Action**:
- If count is 0–5: report count only, no action required.
- If count > 5: recommend a reclassification session. List the `_inbox` files with their titles (from frontmatter or filename). Do not reclassify automatically.

---

### Check 8 — Frontmatter Validity

**Definition**: Every page outside `_inbox/` must have complete, valid frontmatter.

**Procedure**:
For each page (excluding `_inbox/` drafts), verify:
1. All required fields are present: `id`, `title`, `project`, `type`, `status`, `tags`, `refs`, `related`, `created`, `updated`.
2. `id` value matches the filename slug (filename without `.md` extension).
3. `type` is one of the 6 valid values: `adr`, `module`, `glossary`, `runbook`, `incident`, `changelog`.
4. `status` is valid for the page's `type` (see status table in Reference section).
5. `project` field matches the project directory name.

**Action**:
- Auto-fix: add missing fields with empty/default values (`tags: []`, `refs: []`, `related: []`). Report all additions.
- Flag for user: incorrect `type`, invalid `status`, `id`/filename mismatch, `project` field mismatch. These require human judgment to correct.

---

## Auto-fix vs User Approval Matrix

| Issue                                       | Resolution      |
|---------------------------------------------|-----------------|
| Missing wikilinks (unambiguous match)       | Auto-fix        |
| Missing frontmatter fields (empty default)  | Auto-fix        |
| `index.md` missing new pages                | Auto-fix        |
| `index.md` entries pointing to deleted files| Auto-fix        |
| Module page `status` → `outdated`           | Auto-fix        |
| Orphan deletion or archiving                | User approval   |
| ADR status changes                          | User approval   |
| ADR-code contradiction resolution           | User approval   |
| Superseded ADR chain repair                 | User approval   |
| Ambiguous cross-reference linking           | User approval   |
| Invalid `type` / `status` correction        | User approval   |

**Rule**: No page may be deleted without explicit user approval. Auto-fixes must always be reported with a complete change list before being applied (or reported immediately after application).

---

## Log Entry Format

Append the following block to `project-wiki/<project-name>/log.md` after every lint run:

```markdown
## [YYYY-MM-DD HH:MM] lint | <project>
findings:
  orphans: N
  adr_contradictions: N
  stale_modules: N
  missing_xrefs: N
  superseded_chains: N
  index_mismatch: N
  inbox_backlog: N
  frontmatter_errors: N
actions:
  - auto-fixed: <summary of auto-fix changes, or "none">
  - pending_user: <summary of items awaiting user decision, or "none">
suggestions:
  - <next action 1>
  - <next action 2>
```

- The prefix `## [YYYY-MM-DD HH:MM] lint | <project>` must never be modified or reformatted.
- Append only — never edit or delete previous log entries.

---

## Report Template

Present the following report to the user after all 8 checks complete:

```
# Project Lint Report — <project> ([YYYY-MM-DD HH:MM])

## Summary
| Category           | Count | Auto-fixed | Pending |
|--------------------|-------|------------|---------|
| Orphans            |       |            |         |
| ADR contradictions |       |            |         |
| Stale modules      |       |            |         |
| Missing xrefs      |       |            |         |
| Superseded chains  |       |            |         |
| Index mismatch     |       |            |         |
| _inbox backlog     |       |            |         |
| Frontmatter errs   |       |            |         |

## Auto-fixed
- <list each change made automatically, or "None">

## Needs your decision
- <list each item requiring user judgment, or "None">

## Suggestions
- <at least 1 actionable suggestion based on findings>
```

---

## Hard Rules

1. **One project at a time.** Never run lint across multiple projects in a single execution.
2. **No page deletion without user approval.** Orphan archiving and any destructive action must be user-confirmed.
3. **Auto-fixes must be reported.** Every automated change must appear in the "Auto-fixed" section of the report and in the log entry.
4. **All 8 checks must run.** Even if a check finds 0 issues, it must be recorded in the summary table with a count of 0.
5. **Log prefix format is immutable.** The `## [YYYY-MM-DD HH:MM] lint | <project>` format must never be altered.
6. **At least 1 suggestion must be provided.** The Suggestions section may not be left empty.

---

## Verification Checklist

Before closing the lint session, confirm all of the following:

- [ ] Target project explicitly confirmed by user
- [ ] All 8 checks executed and results recorded
- [ ] Auto-fix changes reported with complete change list
- [ ] User decision items listed separately from auto-fixes
- [ ] `log.md` appended with correctly formatted entry
- [ ] At least 1 suggestion provided in the report
