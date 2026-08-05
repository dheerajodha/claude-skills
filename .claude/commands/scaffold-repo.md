---
description: Assess a repository with AgentReady and apply scaffolding fixes to improve its AI-assisted development readiness score
argument-hint: <repo-path>
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Scaffold Repository for AgentReady

Assess a repository with AgentReady, triage the results, and apply fixes to improve the score.

## Instructions

The repo path is provided in `$ARGUMENTS`. If no argument is provided, prompt the user for the repo path.

Resolve the path to an absolute path and quote it in all shell commands (paths with spaces or special characters will break otherwise). All file operations in subsequent steps should use absolute paths rooted at the repo.

## Step 1: Assess

Run the assessment and capture the JSON output:

```bash
agentready assess "$repo_path"
```

Then parse the JSON results:

```bash
cat "$repo_path/.agentready/assessment-latest.json"
```

Extract from the JSON:
- `overall_score` and `certification_level`
- For each finding: `attribute.id`, `attribute.default_weight`, `score`, `status`, `evidence`, `remediation`

## Step 2: Triage

Calculate recoverable points for each non-passing attribute: `weight × (100 - score)`.

Present a summary table sorted by recoverable points, grouped into three categories:

### Auto-fixable
Changes the skill can apply directly. These are safe, mechanical fixes:

| Attribute | Fix |
|---|---|
| `gitignore_completeness` | Append missing patterns identified in the assessment evidence |
| `conventional_commits` | Add `.commitlintrc.yaml` + commitlint hook to `.pre-commit-config.yaml` (create the file if missing) |
| `issue_pr_templates` | Add `.github/pull_request_template.md` with What/Why/Tickets sections; add issue templates if missing |
| `deterministic_enforcement` | Add `.claude/settings.json` with a PostToolUse formatting hook for the repo's primary language |
| `readme_structure` | Add missing sections (Installation/Getting Started) if the assessment evidence says they're absent |

### Guided
Changes that need user input or repo-specific knowledge. Do not generate content without asking:

| Attribute | Guidance |
|---|---|
| `single_file_verification` | Suggest lint + type-check commands based on the detected language; ask the user to confirm the right commands before adding to AGENTS.md |
| `pattern_references` | Ask the user for 3-5 common change types and their reference files, then add to AGENTS.md |
| `progressive_disclosure` | Suggest path-scoped `.claude/rules/` files based on the repo's directory structure; confirm with user |
| `design_intent` | Explain what's needed (preconditions, invariants, rationale docs); suggest creating a Jira story rather than generating content |
| `threat_model` | Explain the 8-section schema; suggest creating a Jira story |
| `architecture_decisions` | Explain what ADRs are and when they're worth creating; suggest bundling with design_intent story |

### Skip
Attributes that aren't worth fixing — assessor limitations, disproportionate effort, or not applicable. Explain why for each.

Ask the user which categories to proceed with.

## Step 3: Apply auto-fixes

For each auto-fixable attribute the user approved, apply the fix. Explain each change as you make it.

**Idempotency rule:** Before writing any file, check if it already exists. If it does, read it and merge rather than overwrite — do not duplicate entries in `.gitignore`, do not add a second `repos:` key to `.pre-commit-config.yaml`, do not clobber existing `.claude/settings.json` hooks or permissions. If a file already contains the target content, skip it.

### gitignore_completeness
Read the assessment evidence to find which patterns are missing. Check the existing `.gitignore` and only append patterns that are not already covered (including by glob patterns). Group by category (build artifacts, editor files, OS files).

### conventional_commits
If `.commitlintrc.yaml` already exists, skip. Otherwise create it:
```yaml
extends:
  - "@commitlint/config-conventional"
```

If `.pre-commit-config.yaml` exists, check whether it already has a commitlint hook before appending. If it doesn't exist, create it with the repo's existing license header style:
```yaml
repos:
  - repo: https://github.com/alessandrojcm/commitlint-pre-commit-hook
    rev: v9.26.0
    hooks:
      - id: commitlint
        stages: [commit-msg]
        additional_dependencies: ["@commitlint/config-conventional"]
```

### issue_pr_templates
Add `.github/pull_request_template.md`:
```markdown
#### What:
<!--- What is this change doing? --->

#### Why:
<!--- Please include the context and background for your change. --->

#### Tickets:
<!--- Please link to any related Jira issue here. --->
```

If no issue templates exist (check `.github/ISSUE_TEMPLATE/`), add a bug report template, feature request template, and config.yaml with `blank_issues_enabled: true`.

### deterministic_enforcement
Create `.claude/settings.json` with a PostToolUse hook appropriate to the detected language:

- **Go**: run `gofmt` on edited `.go` files
- **Python**: run `black` or `ruff format` on edited `.py` files
- **Rego**: run `make fmt` on edited `.rego` files
- **JavaScript/TypeScript**: run `prettier` on edited files
- **Other**: ask the user what formatter to use

### readme_structure
Read the assessment evidence to identify missing sections. Add only the sections the assessor flagged as missing. Keep additions minimal — a heading, a sentence or two, and a command. Do not rewrite the README.

## Step 4: Guided fixes

For each guided attribute the user wants to address, follow the guidance in the table above. Always ask before writing content. For design_intent, threat_model, and architecture_decisions, suggest creating Jira stories with structured descriptions rather than generating the content directly.

## Step 5: Re-assess and report

Run `agentready assess "$repo_path"` again. Parse the new JSON and compare every attribute against the original:

```text
Before: <score>/100 (<level>)
After:  <score>/100 (<level>)

Improved:
  <attribute>: <old_score> → <new_score> (+<delta> pts)
  ...

Regressed:
  <attribute>: <old_score> → <new_score> (-<delta> pts, <reason>)
  ...

Still failing:
  <attribute>: <score>/100 (<reason it wasn't fixed>)
  ...
```

If any attributes regressed, investigate immediately — a scaffolding fix may have broken existing config.

If the score hasn't reached Gold (75), suggest what remaining changes would get there and the effort involved.
