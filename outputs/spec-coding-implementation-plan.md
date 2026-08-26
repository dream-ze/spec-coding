# Personal Cross-Agent Spec Coding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a portable `spec-coding` package that installs one policy Skill for Codex and Claude, classifies work into Quick, Feature Spec, or Complex Bug Spec, and safely bootstraps both Spec Kit integrations in an explicitly selected project.

**Architecture:** A shared Markdown Skill is the policy and orchestration layer. Two Python 3.9 standard-library scripts handle personal Skill installation and per-project Spec Kit bootstrap; both default to read-only planning and require `--apply` before writing. Automated `unittest` coverage verifies package integrity, idempotent installation, integration planning, command execution, and local Git exclusion behavior.

**Tech Stack:** Markdown/Agent Skills, Python 3.9 standard library, `unittest`, Git CLI, GitHub Spec Kit CLI command contracts verified against v1.0.1.

**Spec:** `outputs/spec-coding-personal-cross-agent-design.md`

## Global Constraints

- The package is personal and supports only Codex and Claude in the MVP.
- Feature development and complex defects require Spec; only safe localized work may use Quick.
- A task already classified as Spec must not be silently downgraded.
- Quick still requires a short design and explicit approval before source edits.
- Never install third-party dependencies automatically.
- Never modify a project's `.gitignore`; use `.git/info/exclude` only after `--apply`.
- Never stage, commit, push, or initialize a company repository.
- The current projectless workspace is not a Git repository. Do not create a repository solely to manufacture commit checkpoints; each task's passing focused test suite is its review checkpoint.
- All Python must run on the available Python 3.9.6 without third-party packages.
- Before Task 1 implementation, read and follow `skill-creator` and `superpowers:writing-skills`; before claiming completion, use `superpowers:verification-before-completion`.

---

## File map

| Path | Responsibility |
| --- | --- |
| `outputs/spec-coding/README.md` | Installation, invocation, bootstrap, safety, and verification guide. |
| `outputs/spec-coding/skill/SKILL.md` | Agent-neutral entry contract, routing rule, approval gates, and reference routing. |
| `outputs/spec-coding/skill/references/classification.md` | Exact Quick/Feature/Complex Bug classification criteria and upgrade rules. |
| `outputs/spec-coding/skill/references/quick-path.md` | Short-design, approval, implementation, and verification flow for Quick work. |
| `outputs/spec-coding/skill/references/spec-path.md` | Feature and complex-bug flows, Spec Kit delegation, and Markdown fallback contract. |
| `outputs/spec-coding/scripts/install.py` | Dry-run-first installation of the canonical Skill into personal Codex and Claude locations. |
| `outputs/spec-coding/scripts/bootstrap_project.py` | Dry-run-first Spec Kit integration planning, execution, and `.git/info/exclude` maintenance. |
| `outputs/spec-coding/tests/test_skill_package.py` | Static Skill metadata/reference/content validation. |
| `outputs/spec-coding/tests/test_install.py` | Installer planning, conflict, replacement-backup, and idempotency tests. |
| `outputs/spec-coding/tests/test_bootstrap.py` | Bootstrap state parsing, command planning/execution, path safety, and exclude tests. |
| `outputs/spec-coding/tests/scenarios.json` | Representative prompt-to-route acceptance matrix. |
| `outputs/spec-coding/tests/manual-smoke.md` | Repeatable Codex and Claude smoke-test record template. |

---

### Task 1: Canonical Skill and package contract

**Files:**
- Create: `outputs/spec-coding/skill/SKILL.md`
- Create: `outputs/spec-coding/skill/references/classification.md`
- Create: `outputs/spec-coding/skill/references/quick-path.md`
- Create: `outputs/spec-coding/skill/references/spec-path.md`
- Create: `outputs/spec-coding/tests/test_skill_package.py`
- Create: `outputs/spec-coding/tests/scenarios.json`

**Interfaces:**
- Consumes: Approved decisions in `outputs/spec-coding-personal-cross-agent-design.md`.
- Produces: A canonical `skill/` directory accepted by both agents; relative references `references/classification.md`, `references/quick-path.md`, and `references/spec-path.md`; scenario labels `quick`, `feature_spec`, `complex_bug_spec`.

- [ ] **Step 1: Write the failing package-integrity tests**

Create `outputs/spec-coding/tests/test_skill_package.py` with these checks:

```python
import json
import re
import unittest
from pathlib import Path


ROOT = Path(__file__).resolve().parents[1]
SKILL = ROOT / "skill" / "SKILL.md"
REFERENCES = ROOT / "skill" / "references"
SCENARIOS = ROOT / "tests" / "scenarios.json"


class SkillPackageTests(unittest.TestCase):
    def test_skill_has_required_frontmatter(self):
        text = SKILL.read_text(encoding="utf-8")
        match = re.match(r"^---\n(.*?)\n---\n", text, re.DOTALL)
        self.assertIsNotNone(match)
        frontmatter = match.group(1)
        self.assertIn("name: spec-coding", frontmatter)
        self.assertIn("description:", frontmatter)
        self.assertIn("Quick", frontmatter)
        self.assertIn("Spec", frontmatter)

    def test_all_declared_references_exist(self):
        text = SKILL.read_text(encoding="utf-8")
        declared = set(re.findall(r"references/[a-z0-9-]+\.md", text))
        self.assertEqual(
            declared,
            {
                "references/classification.md",
                "references/quick-path.md",
                "references/spec-path.md",
            },
        )
        for relative in declared:
            self.assertTrue((ROOT / "skill" / relative).is_file(), relative)

    def test_skill_contains_non_bypassable_gates(self):
        text = SKILL.read_text(encoding="utf-8")
        self.assertIn("announce the classification", text)
        self.assertIn("Do not edit source code before approval", text)
        self.assertIn("must stop and upgrade", text)

    def test_reference_contracts_are_present(self):
        classification = (REFERENCES / "classification.md").read_text(encoding="utf-8")
        quick = (REFERENCES / "quick-path.md").read_text(encoding="utf-8")
        spec = (REFERENCES / "spec-path.md").read_text(encoding="utf-8")
        self.assertIn("public API", classification)
        self.assertIn("authentication", classification)
        self.assertIn("short design", quick)
        self.assertIn("focused verification", quick)
        self.assertIn("speckit-specify", spec)
        self.assertIn("speckit-bug-assess", spec)
        self.assertIn("Markdown fallback", spec)

    def test_scenario_matrix_has_all_routes(self):
        scenarios = json.loads(SCENARIOS.read_text(encoding="utf-8"))
        routes = {item["expected_route"] for item in scenarios}
        self.assertEqual(routes, {"quick", "feature_spec", "complex_bug_spec"})
        self.assertTrue(all(item["prompt"].strip() for item in scenarios))


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run the focused tests and confirm RED**

Run:

```bash
python3 -m unittest outputs/spec-coding/tests/test_skill_package.py -v
```

Expected: errors reporting missing `skill/SKILL.md`, reference files, or `scenarios.json`.

- [ ] **Step 3: Create the canonical `SKILL.md`**

Create `outputs/spec-coding/skill/SKILL.md` with this exact structure and policy:

```markdown
---
name: spec-coding
description: Classify personal Codex or Claude development work into Quick, Feature Spec, or Complex Bug Spec; enforce a design approval before edits; and orchestrate GitHub Spec Kit when available. Use for requests to build features, change behavior, fix bugs, edit configuration, or explicitly invoke spec-first/quick-fix. Do not use for read-only explanation or review.
---

# Spec Coding

Classify before implementation. Read `references/classification.md`, announce the classification and evidence, and wait at every approval gate defined by the selected path.

## Hard gates

1. Always inspect the project before classifying.
2. Always announce the classification before proposing implementation.
3. Do not edit source code before approval of the selected path's design or remediation.
4. A user may force Spec. A Quick request that violates the Quick criteria must be upgraded.
5. If hidden complexity appears during Quick work, you must stop and upgrade before further edits.
6. Preserve unrelated user changes. Never discard or overwrite them without explicit permission.
7. Never install Spec Kit, initialize a project, stage artifacts, or commit them without explicit permission for that action.

## Route

- `quick`: read and follow `references/quick-path.md`.
- `feature_spec`: read and follow the Feature section of `references/spec-path.md`.
- `complex_bug_spec`: read and follow the Complex Bug section of `references/spec-path.md`.

Use Spec Kit skills when present. If they are absent, use the Markdown fallback contract in `references/spec-path.md` and label the fallback explicitly.
```

- [ ] **Step 4: Create the three focused reference files**

Write `classification.md` with an all-criteria Quick rule, explicit Feature and Complex Bug risk signals, and the no-silent-downgrade rule from the approved design. Include public API, persisted data, schema, authentication/authorization, cross-service contracts, uncertain root cause, intermittent failures, security, data loss, concurrency, reliability, and significant performance risk as disqualifiers for Quick.

Write `quick-path.md` with this exact sequence:

```text
inspect → announce Quick with evidence → short design + focused verification
→ explicit approval → narrow edit → focused verification → evidence report
```

State that Quick creates no persistent Spec artifact by default and must stop on scope expansion.

Write `spec-path.md` with the exact delegation order:

```text
Feature:
speckit-specify → speckit-clarify → user requirements approval
→ speckit-plan → speckit-tasks → speckit-analyze
→ user implementation approval → scoped speckit-implement
→ verification → speckit-converge

Complex Bug:
speckit-bug-assess → user remediation approval
→ speckit-bug-fix → speckit-bug-test
```

Include a Markdown fallback with `specs/<slug>/spec.md`, `plan.md`, `tasks.md`, and `verification.md` for Feature work, and `.specify/bugs/<slug>/assessment.md`, `fix.md`, and `test.md` for Complex Bug work.

- [ ] **Step 5: Create the scenario matrix**

Create `outputs/spec-coding/tests/scenarios.json`:

```json
[
  {"id":"new-endpoint","prompt":"Add a new public endpoint for cluster statistics.","expected_route":"feature_spec"},
  {"id":"prod-timeout","prompt":"Fix an intermittent production timeout with no known root cause.","expected_route":"complex_bug_spec"},
  {"id":"config-value","prompt":"Correct one documented timeout value in the local development config.","expected_route":"quick"},
  {"id":"schema-upgrade","prompt":"Use quick-fix, but the change requires a persisted schema migration.","expected_route":"feature_spec"},
  {"id":"forced-spec","prompt":"Use spec-first to fix a typo.","expected_route":"feature_spec"},
  {"id":"auth-upgrade","prompt":"Use quick-fix to change authentication token validation.","expected_route":"complex_bug_spec"}
]
```

- [ ] **Step 6: Run the focused tests and confirm GREEN**

Run:

```bash
python3 -m unittest outputs/spec-coding/tests/test_skill_package.py -v
```

Expected: 5 tests pass.

---

### Task 2: Dry-run-first personal Skill installer

**Files:**
- Create: `outputs/spec-coding/scripts/install.py`
- Create: `outputs/spec-coding/tests/test_install.py`

**Interfaces:**
- Consumes: Canonical directory `outputs/spec-coding/skill` from Task 1.
- Produces: `InstallAction(action: str, source: Path, destination: Path, backup: Path | None)`; `plan_install(...)`; `apply_install(...)`; CLI flags `--codex-root`, `--claude-root`, `--apply`, and `--replace`.

- [ ] **Step 1: Write failing installer tests**

Create `outputs/spec-coding/tests/test_install.py` that loads `scripts/install.py` with `importlib.util` and covers:

```python
import importlib.util
import sys
import tempfile
import unittest
from pathlib import Path


ROOT = Path(__file__).resolve().parents[1]
SCRIPT = ROOT / "scripts" / "install.py"


def load_module():
    spec = importlib.util.spec_from_file_location("spec_coding_install", SCRIPT)
    module = importlib.util.module_from_spec(spec)
    sys.modules[spec.name] = module
    spec.loader.exec_module(module)
    return module


class InstallTests(unittest.TestCase):
    def setUp(self):
        self.module = load_module()
        self.temp = tempfile.TemporaryDirectory()
        self.root = Path(self.temp.name)
        self.source = self.root / "source"
        self.source.mkdir()
        (self.source / "SKILL.md").write_text("canonical\n", encoding="utf-8")

    def tearDown(self):
        self.temp.cleanup()

    def test_plan_creates_both_agent_destinations(self):
        actions = self.module.plan_install(
            self.source,
            [self.root / "codex" / "spec-coding", self.root / "claude" / "spec-coding"],
            replace=False,
        )
        self.assertEqual([item.action for item in actions], ["create", "create"])

    def test_apply_is_idempotent(self):
        destinations = [self.root / "codex" / "spec-coding", self.root / "claude" / "spec-coding"]
        self.module.apply_install(self.module.plan_install(self.source, destinations, False))
        second = self.module.plan_install(self.source, destinations, False)
        self.assertEqual([item.action for item in second], ["unchanged", "unchanged"])

    def test_conflict_requires_replace(self):
        destination = self.root / "codex" / "spec-coding"
        destination.mkdir(parents=True)
        (destination / "SKILL.md").write_text("user version\n", encoding="utf-8")
        with self.assertRaisesRegex(ValueError, "--replace"):
            self.module.plan_install(self.source, [destination], replace=False)

    def test_replace_preserves_backup(self):
        destination = self.root / "codex" / "spec-coding"
        destination.mkdir(parents=True)
        (destination / "SKILL.md").write_text("user version\n", encoding="utf-8")
        actions = self.module.plan_install(self.source, [destination], replace=True)
        self.module.apply_install(actions)
        self.assertEqual((destination / "SKILL.md").read_text(), "canonical\n")
        self.assertTrue(actions[0].backup.is_dir())
        self.assertEqual((actions[0].backup / "SKILL.md").read_text(), "user version\n")
```

- [ ] **Step 2: Run the tests and confirm RED**

Run:

```bash
python3 -m unittest outputs/spec-coding/tests/test_install.py -v
```

Expected: import failure because `scripts/install.py` does not exist.

- [ ] **Step 3: Implement directory comparison and install planning**

In `install.py`, implement:

```python
@dataclass(frozen=True)
class InstallAction:
    action: str
    source: Path
    destination: Path
    backup: Optional[Path] = None


def directories_equal(left: Path, right: Path) -> bool:
    left_files = sorted(path.relative_to(left) for path in left.rglob("*") if path.is_file())
    right_files = sorted(path.relative_to(right) for path in right.rglob("*") if path.is_file())
    if left_files != right_files:
        return False
    return all((left / rel).read_bytes() == (right / rel).read_bytes() for rel in left_files)
```

`plan_install(source, destinations, replace)` must return `create`, `unchanged`, or `replace` actions. A different existing destination raises `ValueError` unless `replace=True`. A replace action chooses a non-existing sibling backup named `spec-coding.backup-YYYYMMDD-HHMMSS`, adding `-2`, `-3`, and so on if necessary.

- [ ] **Step 4: Implement safe application and CLI behavior**

`apply_install(actions)` must:

- create missing parent directories;
- use `shutil.copytree` for `create`;
- do nothing for `unchanged`;
- move a conflicting directory to the planned backup before copying the canonical directory;
- if copying after a move fails, move the backup back into place before re-raising.

The CLI defaults to these destinations:

```text
~/.agents/skills/spec-coding
~/.claude/skills/spec-coding
```

Without `--apply`, print the plan and exit 0 without writes. `--replace` only authorizes backup-and-replace; it does not imply `--apply`.

- [ ] **Step 5: Run focused installer tests**

Run:

```bash
python3 -m unittest outputs/spec-coding/tests/test_install.py -v
```

Expected: 4 tests pass.

- [ ] **Step 6: Verify CLI dry-run against temporary destinations**

Run:

```bash
tmp_dir="$(mktemp -d)"
python3 outputs/spec-coding/scripts/install.py \
  --codex-root "$tmp_dir/codex" \
  --claude-root "$tmp_dir/claude"
test ! -e "$tmp_dir/codex/spec-coding"
test ! -e "$tmp_dir/claude/spec-coding"
```

Expected: the script prints two `create` actions and both `test` commands succeed.

---

### Task 3: Project bootstrap planning and Git-local exclusion

**Files:**
- Create: `outputs/spec-coding/scripts/bootstrap_project.py`
- Create: `outputs/spec-coding/tests/test_bootstrap.py`

**Interfaces:**
- Consumes: Project root, optional `--with-bug-extension`, optional `--specify-command`.
- Produces: `BootstrapPlan(project_root, commands, exclude_entries, warnings)`; `read_integration_state(...)`; `plan_bootstrap(...)`; `update_info_exclude(...)`.

- [ ] **Step 1: Write failing state and exclusion tests**

Create `test_bootstrap.py` with the same `importlib.util` loader pattern used in `test_install.py`, including registration in `sys.modules` before `exec_module`, and tests that:

1. parse `.specify/integration.json` with `installed_integrations: ["codex"]`;
2. plan only `integration install claude` when Codex is already installed;
3. plan `init` followed by Claude install for an uninitialized project;
4. add optional `extension add bug` only when requested and not registered;
5. preserve existing `.git/info/exclude` content and avoid duplicate entries;
6. reject a project path that is inside, but not equal to, its Git root.

Use this expected init command tuple:

```python
(
    "specify", "init", "--here", "--force", "--non-interactive",
    "--ignore-agent-tools", "--script", "sh", "--integration", "codex",
)
```

Use these exclusion entries:

```python
DEFAULT_EXCLUDES = (
    "/.specify/",
    "/specs/",
    "/.agents/skills/speckit-*/",
    "/.claude/skills/speckit-*/",
)
```

- [ ] **Step 2: Run the bootstrap tests and confirm RED**

Run:

```bash
python3 -m unittest outputs/spec-coding/tests/test_bootstrap.py -v
```

Expected: import failure because `bootstrap_project.py` does not exist.

- [ ] **Step 3: Implement safe state readers**

Implement `read_integration_state(project_root)` to read `.specify/integration.json` as UTF-8 JSON. Return an empty installed set when the file is absent. Raise `ValueError` when JSON is invalid, is not an object, or declares `installed_integrations` as a non-list. Normalize only non-empty string keys.

Implement `bug_extension_installed(project_root)` by reading `.specify/extensions/registry.json` and checking whether the `extensions` object contains key `bug`. Treat a missing registry as not installed and malformed content as a loud `ValueError`.

- [ ] **Step 4: Implement project-root validation and planning**

`resolve_project_root(project)` must:

- require an existing directory;
- run `git -C <project> rev-parse --show-toplevel` when Git metadata is available;
- require the resolved user path to equal the resolved Git root;
- allow a non-Git directory but add a warning that `.git/info/exclude` is unavailable.

`plan_bootstrap(...)` must use `.specify/integration.json` rather than parsing human-readable `specify integration status` output. Plan:

- init command when no integration state exists;
- `specify integration install codex` when state exists without Codex;
- `specify integration install claude` when state exists without Claude;
- `specify extension add bug` only when requested and absent;
- no command for an already installed component.

- [ ] **Step 5: Implement idempotent `.git/info/exclude` updates**

`update_info_exclude(git_dir, entries)` must append this marked block only when entries are missing:

```text

# spec-coding personal artifacts
/.specify/
/specs/
/.agents/skills/speckit-*/
/.claude/skills/speckit-*/
```

Preserve every existing byte before the appended block, ensure one final newline, and never duplicate an entry on subsequent calls. Resolve the actual Git directory with `git -C <root> rev-parse --git-dir`; do not assume `.git` is a directory because worktrees may use a `.git` file.

- [ ] **Step 6: Run focused bootstrap tests**

Run:

```bash
python3 -m unittest outputs/spec-coding/tests/test_bootstrap.py -v
```

Expected: all state, planning, exclusion, and root-safety tests pass.

---

### Task 4: Bootstrap execution, dry-run CLI, and rollback reporting

**Files:**
- Modify: `outputs/spec-coding/scripts/bootstrap_project.py`
- Modify: `outputs/spec-coding/tests/test_bootstrap.py`

**Interfaces:**
- Consumes: `BootstrapPlan` from Task 3.
- Produces: `CommandResult(argv, returncode, stdout, stderr)`; `run_plan(...)`; CLI flags `--project`, `--with-bug-extension`, `--specify-command`, and `--apply`.

- [ ] **Step 1: Add failing execution tests**

Extend `test_bootstrap.py` with a temporary executable named `fake-specify` that appends its arguments to a log file and exits 0. Verify:

- dry-run executes no external commands and does not change `.git/info/exclude`;
- `--apply` executes commands in plan order with `cwd=project_root`;
- a nonzero command stops later commands and leaves the exclude file unchanged;
- successful commands update `.git/info/exclude` only after all commands pass;
- missing `specify` produces exit code 2 and an installation instruction, not an automatic install.

- [ ] **Step 2: Run the new tests and confirm RED**

Run:

```bash
python3 -m unittest outputs/spec-coding/tests/test_bootstrap.py -v
```

Expected: failures for missing `run_plan` and CLI behavior.

- [ ] **Step 3: Implement command execution**

Implement:

```python
@dataclass(frozen=True)
class CommandResult:
    argv: Tuple[str, ...]
    returncode: int
    stdout: str
    stderr: str


def run_command(argv: Sequence[str], cwd: Path) -> CommandResult:
    completed = subprocess.run(
        list(argv), cwd=str(cwd), text=True, capture_output=True, check=False
    )
    return CommandResult(tuple(argv), completed.returncode, completed.stdout, completed.stderr)
```

`run_plan` must stop on the first nonzero result and return the completed results plus a failure summary. It must not run delete, uninstall, or cleanup commands. After a partial failure, print which commands succeeded and the exact failed command; preserve all created files for inspection.

- [ ] **Step 4: Implement the dry-run-first CLI**

Behavior:

- `--project` is required and displayed as an absolute resolved path;
- `--specify-command` defaults to `specify` and is resolved with `shutil.which`;
- without `--apply`, print warnings, command plan, and exclusion plan, then exit 0;
- with `--apply`, require the resolved command to exist, execute the plan, and update the exclude file only on full success;
- return 0 on success, 1 on command failure or unsafe state, and 2 when `specify` is missing;
- the missing-tool message recommends `uv tool install specify-cli` but never executes it.

- [ ] **Step 5: Run bootstrap tests and an end-to-end fake CLI smoke test**

Run:

```bash
python3 -m unittest outputs/spec-coding/tests/test_bootstrap.py -v
tmp_repo="$(mktemp -d)"
git -C "$tmp_repo" init -q
python3 outputs/spec-coding/scripts/bootstrap_project.py --project "$tmp_repo"
test ! -e "$tmp_repo/.specify"
```

Expected: tests pass; dry-run prints the init/Claude plan; no `.specify` directory is created.

---

### Task 5: User documentation, smoke-test protocol, and full verification

**Files:**
- Create: `outputs/spec-coding/README.md`
- Create: `outputs/spec-coding/tests/manual-smoke.md`
- Modify: `outputs/spec-coding/tests/test_skill_package.py`

**Interfaces:**
- Consumes: Completed Skill and scripts from Tasks 1–4.
- Produces: A self-contained prototype package and repeatable manual verification instructions.

- [ ] **Step 1: Extend static tests for documentation coverage**

Add assertions that `README.md` contains:

- prerequisites: Python 3.9+, Codex or Claude, optional `specify`;
- dry-run and `--apply` examples for both scripts;
- explicit invocation examples `$spec-coding` for Codex and `/spec-coding` or Skill selection for Claude, while noting that the exact Claude UI invocation may vary;
- Quick, Feature Spec, and Complex Bug Spec examples;
- a statement that live installation and project bootstrap are not performed automatically;
- uninstall instructions that preserve backups and require the user to choose the exact destination.

Add an assertion that `manual-smoke.md` contains separate Codex and Claude result sections with classification, approval-gate, artifact-sharing, and verdict fields.

- [ ] **Step 2: Run static tests and confirm RED**

Run:

```bash
python3 -m unittest outputs/spec-coding/tests/test_skill_package.py -v
```

Expected: documentation assertions fail because the files are absent.

- [ ] **Step 3: Write the README**

Document this safe evaluation sequence:

```bash
# 1. Inspect the personal install plan only
python3 scripts/install.py

# 2. Install only after reviewing the exact destinations
python3 scripts/install.py --apply

# 3. Inspect one project's Spec Kit bootstrap plan only
python3 scripts/bootstrap_project.py --project /absolute/path/to/project --with-bug-extension

# 4. Apply only after reviewing commands and local excludes
python3 scripts/bootstrap_project.py --project /absolute/path/to/project --with-bug-extension --apply
```

Explain that `--replace` creates a recoverable backup, that bootstrap may merge Spec Kit files into a non-empty repository, and that users should review the dry-run and current Git status first.

- [ ] **Step 4: Write the manual smoke protocol**

Use the same six prompts from `scenarios.json` in both Codex and Claude. For each agent, record:

```text
Agent/version:
Skill invocation method:
Prompt ID:
Observed classification:
Classification evidence shown: yes/no
Implementation blocked before approval: yes/no
Shared project artifact path:
Verdict: pass/partial/fail
Notes:
```

The authentication prompt must be rejected as Quick; the schema prompt must upgrade; the typo forced through `spec-first` must remain Spec.

- [ ] **Step 5: Run the complete automated suite**

Run:

```bash
python3 -m unittest discover -s outputs/spec-coding/tests -p 'test_*.py' -v
```

Expected: all tests pass with no skipped tests.

- [ ] **Step 6: Run syntax and side-effect verification**

Run:

```bash
python3 -m py_compile \
  outputs/spec-coding/scripts/install.py \
  outputs/spec-coding/scripts/bootstrap_project.py

test ! -e "$HOME/.agents/skills/spec-coding"
test ! -e "$HOME/.claude/skills/spec-coding"
```

Expected: compilation succeeds; prototype creation has not installed anything into the live personal Skill locations. If either destination existed before this work, replace the last two assertions with a byte-for-byte before/after checksum comparison and require no change.

- [ ] **Step 7: Package review against the approved spec**

Verify each of the nine MVP acceptance criteria in `outputs/spec-coding-personal-cross-agent-design.md`. Record actual automated test output in the final handoff, label Codex/Claude live smoke tests as not run until the user authorizes live installation, and do not claim those criteria passed prematurely.

---

## Execution completion criteria

Implementation is ready for user evaluation when Tasks 1–5 pass their focused tests, the full `unittest` suite and `py_compile` pass, the installer and bootstrap default to read-only plans, and no live Codex/Claude configuration or existing project has been changed.
