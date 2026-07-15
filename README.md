# Claude EC Skills

Claude Code skills and commands for [Conforma](https://github.com/conforma/cli) policy debugging and sprint management.

## Overview

These tools help you:

- Debug and resolve Conforma/EC validation failures
- Track mid-sprint story additions with structured tagging
- Generate sprint reports surfacing unplanned work

## Installation

Copy the `.claude` directory to your project or home directory:

```bash
# Clone this repo
git clone https://github.com/conforma/claude-skills.git

# Copy to your project
cp -r claude-skills/.claude /path/to/your/project/

# Or copy to home directory for global access
cp -r claude-skills/.claude ~/
```

## Commands

### `/ec-setup`

Set up a local debugging environment from a Conforma validation log.

```bash
/ec-setup logs/validation.log
```

This command:
1. Extracts the policy configuration from the log
2. Saves the public key for signature verification
3. Pulls the policy OCI bundle
4. Generates a `run.sh` script to reproduce the validation locally

### `/ec-debug-violations`

Parse a log file and debug all policy violations.

```bash
/ec-debug-violations logs/validation.log
```

This command:
1. Extracts all unique violation codes from the log
2. Looks up each rule's metadata (title, description, solution)
3. Analyzes affected components and error messages
4. Provides root cause analysis and recommended actions
5. Generates a prioritized summary

### `/mid-sprint-add`

Tag a Jira issue as a mid-sprint addition with a structured comment.

```bash
/mid-sprint-add EC-1234 customer escalation from KFLUXSPRT-7872
```

This command:
1. Looks up the current active sprint for the Conforma team
2. Adds a `[MID-SPRINT-ADD] reason: ...` comment to the issue
3. Moves the issue into the active sprint if not already there

### `/sprint-report`

Generate a report of all mid-sprint story additions.

```bash
/sprint-report          # Current/most recent sprint
/sprint-report --post   # Generate and post to #team-conforma
```

This command:
1. Queries all issues in the sprint
2. Searches comments for `[MID-SPRINT-ADD]` tags
3. Generates a report: which stories were added, by whom, when, and why
4. Optionally posts to the team Slack channel

## Skills

### `ec-policy-debugging`

Core skill for investigating individual policy violations. Automatically invoked when asking about EC violations.

**Example prompts:**
- "Why did `olm.unmapped_references` fail?"
- "What does the `rpm_packages.unique_version` rule check?"
- "Debug this EC validation error: [paste error]"

### `mid-sprint-tracking`

Track stories added to sprints after they've started. Provides the comment convention and configuration used by `/mid-sprint-add` and `/sprint-report`.

## File Structure

```text
.claude/
├── commands/
│   ├── ec-setup.md              # /ec-setup command
│   ├── ec-debug-violations.md   # /ec-debug-violations command
│   ├── mid-sprint-add.md        # /mid-sprint-add command
│   └── sprint-report.md         # /sprint-report command
├── reports/                     # Generated sprint reports
├── skills/
│   ├── ec-policy-debugging/
│   │   ├── SKILL.md             # Skill definition
│   │   ├── debugging.md         # Full debugging reference
│   │   └── summarize_violations.py  # Log parsing utility
│   └── mid-sprint-tracking/
│       └── SKILL.md             # Mid-sprint tracking skill
└── settings.local.json          # Claude Code settings
```

## Requirements

- [Claude Code](https://claude.ai/claude-code) CLI
- [ec-cli](https://github.com/conforma/cli) (for local validation)
- [conftest](https://www.conftest.dev/) (for pulling policy bundles)
- [cosign](https://github.com/sigstore/cosign) (for downloading attestations)
- [crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane) (for inspecting images)

## Example Workflow

1. **Get a validation log** from a failed Konflux/Conforma pipeline

2. **Set up debugging environment:**
   ```bash
   /ec-setup logs/my-validation.log
   ```

3. **Debug all violations:**
   ```bash
   /ec-debug-violations logs/my-validation.log
   ```

4. **Investigate specific violations:**
   ```text
   "Why is olm.unmapped_references failing for the operator bundle?"
   ```

5. **Run validation locally** (after setup):
   ```bash
   cd release-policies-myimage-amd64/
   ./run.sh
   ```

## License

Apache 2.0
