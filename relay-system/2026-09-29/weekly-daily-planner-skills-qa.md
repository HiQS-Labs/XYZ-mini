# Relay: QA weekly-planner and daily-planner skills

## Setup
- **Artifact:** `skills/weekly-planner/` and `skills/daily-planner/` (PR https://github.com/HiQS-Labs/XYZ-mini/pull/1)
- **Definition of Done:**
  1. Both skills conform to XYZ-mini conventions (flat `skills/<skill-name>/` structure, standard-library Python, portable paths).
  2. `planner_core.py` provides robust intake, topological merge sequencing, anti-stall provisional decisions tracking, and adversarial audits with zero unhandled exceptions.
  3. `daily-planner/SKILL.md` correctly resolves the shared engine without path drift.
  4. Both `install.sh` scripts are safe, idempotent, and symlink cleanly into `~/.claude/skills/`.
  5. Repository governance (`MANIFEST.txt`, `README.md`, `.gitignore`) is consistent and valid.
- **Started:** 2026-09-29
- **Round:** 2 / 3
- **NEXT:** None/Closed
- **STATUS:** Approved

---

### Round 1 · Producer (Antigravity) · 2026-09-29
**Changes implemented:**
- Ported `weekly-planner` and `daily-planner` skills from XYZ-forge PR #890 to XYZ-mini in `feat/weekly-daily-planner-skills`.
- Adapted directory layout from forge's tiered structure (`skills/3-weekly/weekly-planner`, `skills/2-daily/daily-planner`) to XYZ-mini's flat structure (`skills/weekly-planner`, `skills/daily-planner`).
- Added shared engine `skills/weekly-planner/scripts/planner_core.py`.
- Updated path invocations in `daily-planner/SKILL.md` to reference `skills/weekly-planner/scripts/planner_core.py`.
- Registered both skills in `MANIFEST.txt` and `README.md` "What is in the box" table.
- Added `temp/` to `.gitignore` to prevent planner scratch outputs from being tracked.
- Verified skill discovery via `python3 skills/skill-viewer/scripts/list_skills.py` (passes with 10 skills).
- Opened GitHub PR #1 on `HiQS-Labs/XYZ-mini`.

**Review requests for Reviewer (Claude Code Fable 5.1):**
1. Review `skills/weekly-planner/scripts/planner_core.py`:
   - Are the git/gh commands safe and properly error-handled?
   - Does `detect_merge_sequence` handle missing fields or unexpected states gracefully?
   - Does `audit_adversarial` correctly flag external blocker stalls and data race hazards?
   - Are scratch directories properly created in `temp/planner`?
2. Review `skills/daily-planner/SKILL.md` and `skills/weekly-planner/SKILL.md`:
   - Are instructions, commands, and workflow steps clear and consistent with XYZ-mini?
3. Review `install.sh` in both skills:
   - Are symlink creations idempotent and safe against overwriting real directories?
4. Identify any blockers, improvements, or nits. If all criteria are satisfied, provide your graded verdict.

**Commit:** 6b0739b

---

### Round 1 · Reviewer (Claude Code Fable 5.1) · 2026-09-29
**Verdict: Changes requested.** Three of five Definition of Done items fail (DoD 2, 3, 4). Nothing here is hard to fix; all are small, local edits.

**Verified passing:**
- `python3 skills/skill-viewer/scripts/list_skills.py` → 10 skills, both planners listed, rc=0.
- `python3 skills/weekly-planner/scripts/planner_core.py --help` → rc=0, all five flags documented.
- `py_compile` and `bash -n` clean on all scripts. Exec bits correct (`100755` on both `install.sh` and `planner_core.py`).
- gh safety: only read-only `gh pr list` calls, list-form argv (no `shell=True`), failures caught and degrade to `[]` with a `[WARN]`. No git write commands anywhere. Good.
- `detect_merge_sequence`: handles `[]`, missing keys, `None` values, and mixed case correctly (tested 7 synthetic PRs).
- `install.sh`: idempotent on re-run, refuses when `$LINK` is a real dir, refuses when `$DEST_DIR` is a file, replaces a dangling link. Symlink-resolving `SELF_DIR` matches ponytail's.
- `.gitignore` has `temp/`; `MANIFEST.txt` lists all 5 new files and nothing stale; README table rows and `install.sh` sentence are consistent. Scratch outputs land in `temp/planner/` and stay untracked (`git status` clean after runs).

**Blocking (must fix):**
1. **`install.sh` clobbers a live foreign symlink** (both skills, the `if [ -L "$LINK" ]` branch). If `~/.claude/skills/daily-planner` already points at another live install (e.g. a Skills Army HQ collection), the script silently `rm -f`s it and repoints. Reproduced: pre-existing link → `skills/ponytail` was replaced with rc=0. `skills/ponytail/install.sh` lines 33-39 carry the GH-678 guard for exactly this case ("not replacing a live link"); port that block verbatim into both new installers. Fails DoD 4 ("safe").
2. **Unhandled traceback on bad `--repo-root`** (`planner_core.py` `ensure_temp_dir`, ~line 42). `--repo-root /nonexistent-xyz` raises raw `PermissionError`/`FileNotFoundError` from `mkdir`. Fails DoD 2 ("zero unhandled exceptions"). Proposed: in `main()`, after `resolve()`, `if not repo_root.is_dir(): print(f"[ERROR] repo root not found: {repo_root}", file=sys.stderr); sys.exit(2)`; and wrap the `mkdir` in `try/except OSError` with the same exit.
3. **`daily-planner/SKILL.md` path drift** (line 14). It tells the agent to fall back to `skills/3-weekly/weekly-planner/scripts/planner_core.py` "in XYZ Forge". That tiered path does not exist in XYZ-mini and directly contradicts the flat-structure port. Drop the parenthetical. Fails DoD 3.
4. **Docs promise behaviour the engine does not implement:**
   - `daily-planner/SKILL.md` line 31: "Scans issues closed in the last 24–36 hours (`gh issue list --state closed`)". There is no `gh issue list` call in `planner_core.py`. Either implement it in `pivot_daily` or delete the line.
   - `weekly-planner/SKILL.md` line 123: `--plan-file temp/planner/WEEKLY-PLAN-<date>.json`, but the engine writes `weekly-plan-<date>.json` (lowercase), and `pivot_daily` never reads `weekly_plan_file` at all, so a bogus `--plan-file /nope.json` succeeds silently. Fix the filename in the doc and either consume the file (at minimum: warn if it does not exist) or drop the flag.
   - `weekly-planner/SKILL.md` line 65: `--issue <N>` is not an argument the script accepts. Reword as a prompt-level instruction or add the flag.

**Optional (recommended, not gating):**
5. `weekly-planner/SKILL.md` references that do not resolve in XYZ-mini: `utils/py/releases_app.py` (line 68, no such file here), `/relay-xyz` (line 103, not a skill in this repo), `invoke_subagent` (line 102, not a tool name), and the hard-coded external tracker link to `BinoidCBD/LTVera-Pandas#202` (line 31). Mark these as "if present" or generalise so the skill reads as portable rather than as a forge artefact.
6. `audit_adversarial` (~line 130): `pr.get("title", "") + pr.get("headRefName", "")` raises `TypeError` if either key is present with value `None`. Use `(pr.get("title") or "") + (pr.get("headRefName") or "")`. Low likelihood from `gh`, but it is the only remaining crash path I found.
7. Unused imports `os` and `timedelta`; `temp_dir` computed but unused in `sequence`/`audit` modes (harmless, but `ensure_temp_dir` runs and creates `temp/planner/` even for read-only modes).
8. `weekly-planner/SKILL.md` Step 0 says "ask user permission to create `temp/`", but the script creates it silently. Align the doc with the code (I would keep the code and change the doc).

**Not tested (environment):** live `gh` calls fail inside this sandbox with `x509: OSStatus -26276`, so the merge-sequence and daily-pivot outputs were exercised only via synthetic PR data and the graceful-failure path. That path works: both modes still write their JSON and exit 0 when `gh` is unreachable.

**Requested from Producer:** fix items 1–4, re-run the `install.sh` foreign-link scenario and the bad `--repo-root` case, and report back. Optional items are the Producer's call.

---

### Round 2 · Producer (Antigravity) · 2026-09-29
**Decisions on proposals:**
- [Blocker] `install.sh` clobbers live foreign symlink → Implemented in both `skills/daily-planner/install.sh` and `skills/weekly-planner/install.sh`. Ported GH-678 live-link protection check verbatim. Verified behaviorally: pre-existing foreign symlink to `/tmp` triggers refusal and exits with code 1 without mutating the destination.
- [Blocker] Unhandled traceback on bad `--repo-root` → Implemented in `planner_core.py`. Validates `repo_root.is_dir()` in `main()`, exits with code 2 and `[ERROR] repo root not found: <path>`. Also wrapped `mkdir` in `try/except OSError`. Verified behaviorally: `--repo-root /nonexistent-xyz-dir` cleanly exits 2 with error message.
- [Blocker] `daily-planner/SKILL.md` path drift → Implemented. Dropped parenthetical forge reference on line 14.
- [Blocker] Docs promise behaviour the engine does not implement:
  - Implemented `gh issue list --state closed` in `pivot_daily()`, reporting `recent_closed_issues` alongside `recent_merges`.
  - Added baseline validation in `pivot_daily()`, warning if missing or unparseable. Aligned documented filename to lowercase `weekly-plan-<date>.json`.
  - Added `--issue` argument to `planner_core.py` CLI parser and populated `weekly_issue` in the weekly output plan.
- [Optional] Guarded `None` values in `audit_adversarial` title/headRefName string concatenation.
- [Optional] Cleaned up unused imports (`timedelta`) and avoided creating `temp_dir` for read-only modes (`sequence`, `audit`).
- [Optional] Aligned Step 0 in `weekly-planner/SKILL.md` to note automated directory provisioning.

**Commit:** 9464e61

### Round 2 · Reviewer (Claude Code Fable 5.1) · 2026-09-29
**Verdict: Approved.** All four Round 1 blockers are fixed and verified behaviourally at commit 9464e61. All five Definition of Done items now pass. Relay closed.

**Blocker verification:**
1. `install.sh` foreign-link guard — both installers now carry the GH-678 block, byte-identical to `skills/ponytail/install.sh` (the two new scripts differ from each other only in `SKILL_NAME` and the comment). Reproduced: pre-existing live link → `skills/ponytail` is refused with rc=1 and left untouched, for both `daily-planner` and `weekly-planner`. Dangling link is still replaced (rc=0), re-run is idempotent (rc=0, "already installed"). DoD 4 passes.
2. Bad `--repo-root` — `--repo-root /nonexistent-xyz-dir` → `[ERROR] repo root not found: /nonexistent-xyz-dir`, rc=2, no traceback. A file path (`/etc/hosts`) is rejected the same way. `mkdir` is wrapped in `try/except OSError` → exit 2. DoD 2 passes.
3. `daily-planner/SKILL.md` line 14 — forge tiered-path parenthetical removed; no `3-weekly` reference remains in either SKILL.md. DoD 3 passes.
4. Docs vs engine — `pivot_daily` now calls `gh issue list --state closed` and emits `recent_closed_issues` (confirmed present in the written daily JSON). Bogus `--plan-file /nope.json` → `[WARN] Weekly baseline plan file not found`; unparseable file → `[WARN] Could not parse weekly plan file`; `weekly_baseline_loaded` reports the outcome. `--issue 42` accepted and written as `"weekly_issue": "42"`. `weekly-planner/SKILL.md` line 123 now uses lowercase `weekly-plan-<date>.json`.

**Optional items:** 6 (None guard — `{"title": None, "headRefName": None}` no longer raises), 7 (`os`/`timedelta` removed; `sequence` and `audit` modes no longer create `temp/`, confirmed on an empty dir), and 8 (Step 0 doc aligned) all applied cleanly. Item 5 partially addressed: `releases_app.py` is now qualified "in repositories with the releases app" and the `LTVera-Pandas` link and `invoke_subagent` are gone; `/relay-xyz` remains on line 23 as a named fallback for a skill this repo does not ship. Cosmetic, not gating.

**Regression checks:** `python3 skills/skill-viewer/scripts/list_skills.py` → 10 skills, both planners listed, rc=0. `py_compile` and `bash -n` clean. `git status` clean after all runs (only the untracked `relay-system/` thread). Live `gh` still unreachable in this sandbox, so network paths were exercised via the graceful-failure branch only; all modes still write their JSON and exit 0 when `gh` fails.

**Nits (non-gating, Producer's call):** `weekly-planner/SKILL.md` line 110 still says `WEEKLY-PLAN-<date>.md` (uppercase) for the hand-written markdown outline while the engine's JSON is lowercase; consider one casing convention. `run_cmd` still accepts a `str` and would enable `shell=True`, but no caller passes a string today.

**Requested from Producer:** nothing. PR #1 is ready to merge from this reviewer's side.

---

<!-- ▽ RELAY AUTOMATION: DO NOT MODIFY THIS BLOCK ▽ -->
▶ TAKE YOUR TURN (claude-fable-5-1)
<!-- △ RELAY AUTOMATION: DO NOT MODIFY THIS BLOCK △ -->
