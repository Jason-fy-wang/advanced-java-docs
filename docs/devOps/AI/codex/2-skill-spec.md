---
tags:
  - codex
  - skill
  - agent-skills
  - llm
---
# How To Write A Codex Skill

This note summarizes how to write a high-quality `SKILL.md` for Codex or other Agent Skills compatible clients.

References:

- [Agent Skills specification](https://agentskills.io/specification)
- [Best practices for skill creators](https://agentskills.io/skill-creation/best-practices)

## What Is A Skill?

A skill is a small, self-contained folder that teaches an AI agent how to handle a specific class of tasks.

Use a skill when the agent needs reusable task knowledge, for example:

- A project-specific workflow
- A company coding standard
- A domain-specific review checklist
- A repeatable document, data, or deployment process
- Tool-specific instructions that are easy to forget

Do not use a skill as a general knowledge article. A good skill tells the agent **how to act** in a repeated situation.

## Minimal Folder Structure

```text
skill-name/
|-- SKILL.md          # Required: metadata + instructions
|-- scripts/          # Optional: executable helpers
|-- references/       # Optional: extra documentation loaded on demand
`-- assets/           # Optional: templates, images, config files, examples
```

Only `SKILL.md` is required.

## SKILL.md Required Format

`SKILL.md` must start with YAML front matter, followed by Markdown instructions.

```markdown
---
name: skill-name
description: Describe what this skill does and when the agent should use it.
---

# Skill Name

Follow these instructions when this skill is active.
```

### Required Fields

| Field | Required | Rule |
|---|---:|---|
| `name` | Yes | Max 64 characters. Use lowercase letters, numbers, and hyphens only. |
| `description` | Yes | Max 1024 characters. Explain what the skill does and when to use it. |

The `name` should match the parent folder name.

Good:

```yaml
name: gcp-vpn-runbook
description: Create, validate, and troubleshoot Google Cloud VPN runbooks. Use when writing GCP VPN setup docs, gcloud command guides, or HA VPN troubleshooting instructions.
```

Bad:

```yaml
name: GCPVPN
description: Helps with cloud.
```

Problems in the bad example:

- `name` uses uppercase letters.
- `description` is too vague.
- The trigger condition is unclear.

## Optional Front Matter Fields

```yaml
---
name: pdf-processing
description: Extract PDF text, fill PDF forms, and merge PDF files. Use when working with PDF documents, forms, or document extraction.
license: Apache-2.0
compatibility: Requires Python 3.12+, pypdf, and pdfplumber.
metadata:
  author: platform-team
  version: "1.0"
---
```

Use optional fields only when they help:

- `license`: license name or a bundled license file.
- `compatibility`: environment requirements such as system packages, network access, or product constraints.
- `metadata`: client-specific key-value data.
- `allowed-tools`: experimental field for pre-approved tools. Support depends on the client.

## Progressive Disclosure

Skills should spend context carefully.

Agent Skills use progressive loading:

1. `name` and `description` are always visible to the agent.
2. The full `SKILL.md` body loads only when the skill is triggered.
3. Files in `scripts/`, `references/`, and `assets/` are used only when needed.

Keep `SKILL.md` focused. The Agent Skills specification recommends keeping it under **500 lines** and about **5,000 tokens**.

When a skill becomes large, move details into separate files:

```text
gcp-networking/
|-- SKILL.md
`-- references/
    |-- ha-vpn.md
    |-- cloud-nat.md
    `-- load-balancer.md
```

In `SKILL.md`, tell the agent exactly when to read each file:

```markdown
For HA VPN tasks, read `references/ha-vpn.md`.
For Cloud NAT tasks, read `references/cloud-nat.md`.
For load balancer tasks, read `references/load-balancer.md`.
```

Avoid vague instructions like:

```markdown
Check the references folder for more details.
```

## What To Put In SKILL.md

Include only the information the agent is likely to get wrong without the skill.

Good content:

- Specific workflow steps
- Required commands or scripts
- Project conventions
- Non-obvious edge cases
- Validation steps
- Output format templates
- Common mistakes and how to avoid them

Avoid:

- Generic explanations the model already knows
- Long background theory
- Duplicated content from reference files
- Too many equal options without a clear default
- Human-facing documentation such as `README.md`, `CHANGELOG.md`, or install guides inside the skill folder

## Start From Real Expertise

The best skills come from real work, not generic prompting.

Good sources:

- A completed task where the agent needed corrections
- Internal runbooks
- API specs and schemas
- Code review comments
- Incident reports
- Repeated fixes in version control history

Capture the reusable pattern:

- What steps worked?
- What commands or tools were reliable?
- What mistakes did the agent make?
- What input and output formats matter?
- What project-specific conventions must be followed?

## Calibrate Control

Different tasks need different levels of instruction.

Use high freedom when many approaches are valid:

```markdown
## Code Review

Prioritize security, data loss, race conditions, authorization bugs, and missing tests. Report findings first with file and line references.
```

Use low freedom when the workflow is fragile:

````markdown
## Database Migration

Run exactly this sequence:

```shell
python scripts/migrate.py --verify --backup
```

Do not add flags unless the user explicitly asks.
````

Most skills mix both styles: strict steps for fragile actions, flexible guidance for judgment-based work.

## Prefer Defaults Over Menus

When several tools could work, choose the default. Mention alternatives only as fallbacks.

Good:

```markdown
Use `pdfplumber` for text extraction. If the PDF is scanned and has no embedded text, use OCR with `pdf2image` and `pytesseract`.
```

Bad:

```markdown
You can use pypdf, pdfplumber, PyMuPDF, OCR, or any other PDF library.
```

## Favor Procedures Over Declarations

A skill should teach a reusable process, not one fixed answer.

Good:

```markdown
1. Read the schema from `references/schema.yaml`.
2. Identify tables related to the user request.
3. Join tables using the `_id` foreign key convention.
4. Apply user filters as SQL `WHERE` clauses.
5. Return the result as a Markdown table.
```

Bad:

```markdown
Join `orders` to `customers`, filter `region = 'EMEA'`, and sum `amount`.
```

The bad version only solves one query. The good version generalizes.

## Useful Sections For A Skill

Use only the sections that fit the task.

````markdown
# Skill Name

## When To Use

Use this skill when...

## Workflow

1. Do the first required step.
2. Use the default tool or command.
3. Validate the result.
4. Fix issues and repeat validation if needed.

## Gotchas

- Concrete issue the agent is likely to miss.
- Project-specific naming or path convention.
- Dangerous command or invalid default to avoid.

## Output Format

Use this format:

```markdown
**Findings**
- Severity: file path and line
- Problem
- Recommended fix

**Validation**
- Command run
- Result
```

## References

Read `references/api.md` only when changing API integration behavior.
````

## Gotchas Section

Add a `Gotchas` section when the agent repeatedly makes the same mistake.

Good gotchas are concrete:

```markdown
## Gotchas

- This repo uses `pnpm`, not `npm`.
- Do not edit generated files under `src/generated/`.
- The `/health` endpoint only checks process liveness. Use `/ready` for dependency checks.
```

Avoid generic gotchas:

```markdown
## Gotchas

- Be careful.
- Follow best practices.
- Handle errors.
```

## Validation Loop

For tasks where correctness matters, include a validation loop.

```markdown
## Validation

1. Run `python scripts/validate.py output/`.
2. If validation fails, read the error message and fix the issue.
3. Run validation again.
4. Only finish when validation passes or when the blocker is clearly reported.
```

This is especially useful for:

- Generated documents
- Data transformations
- Migration plans
- Config changes
- Batch edits
- Release or deployment tasks

## When To Use scripts/

Put scripts in `scripts/` when the task needs deterministic behavior.

Good script candidates:

- Repeated parsing logic
- Validation logic
- File conversion
- Report generation
- API calls with exact payload formats
- Batch operations that should not be handwritten each time

Script rules:

- Keep scripts self-contained.
- Print clear error messages.
- Document required arguments.
- Handle edge cases gracefully.
- Let the agent run the script instead of copying large code into context.

Example:

```text
pdf-skill/
|-- SKILL.md
`-- scripts/
    `-- extract_text.py
```

In `SKILL.md`:

```markdown
Use `scripts/extract_text.py input.pdf output.txt` for text extraction.
```

## When To Use references/

Put long documentation in `references/` when it is useful only for some tasks.

Examples:

- API reference
- Database schema
- Policy details
- Framework-specific implementation notes
- Troubleshooting matrix

Keep each reference focused. Link to it from `SKILL.md` with a clear trigger.

```markdown
Read `references/oauth.md` when changing OAuth login, token refresh, or callback handling.
```

## When To Use assets/

Put static files in `assets/` when they are copied, modified, or used as templates.

Examples:

- Document templates
- Image assets
- Config templates
- Example input files
- Boilerplate project files

Do not load assets into the skill body unless the agent needs to inspect them.

## Skill Authoring Checklist

Before publishing a skill, check:

- The folder name matches `name`.
- `name` uses lowercase letters, numbers, and hyphens only.
- `description` says both what the skill does and when to use it.
- `SKILL.md` contains only core instructions.
- Large details are moved to `references/`.
- Repeated deterministic logic is moved to `scripts/`.
- Templates or static resources are moved to `assets/`.
- The skill includes validation steps when output correctness matters.
- The skill avoids generic advice.
- The skill was tested against at least one realistic task.

## Full Example

```text
gcp-runbook-writer/
|-- SKILL.md
|-- references/
|   |-- vpn.md
|   `-- load-balancer.md
`-- scripts/
    `-- check_gcloud_commands.py
```

`SKILL.md`:

```markdown
---
name: gcp-runbook-writer
description: Write Google Cloud operational runbooks with accurate gcloud commands, validation steps, troubleshooting notes, and cleanup commands. Use when creating or reviewing GCP setup docs, cloud networking labs, or deployment runbooks.
compatibility: Requires gcloud CLI knowledge. Network access may be needed to verify current Google Cloud documentation.
---

# GCP Runbook Writer

## Workflow

1. Identify the target GCP service and deployment scenario.
2. Prefer official Google Cloud documentation for command syntax.
3. Write commands with reusable variables for project, region, zone, and resource names.
4. Include verification commands after each major resource group.
5. Include troubleshooting commands for common failure modes.
6. Include cleanup commands for lab environments.

## References

Read `references/vpn.md` for Cloud VPN, HA VPN, Cloud Router, and BGP tasks.
Read `references/load-balancer.md` for Google Cloud load balancer tasks.

## Gotchas

- Use custom VPCs when demonstrating subnet design.
- Do not use overlapping CIDR ranges in VPN examples.
- Include firewall rules when testing VM-to-VM connectivity.
- For HA VPN, include tunnels on both gateway interfaces.

## Output Format

Use Markdown with:

1. Architecture summary
2. Prerequisites
3. Variables
4. Create commands
5. Validation commands
6. Troubleshooting
7. Cleanup
```

## Common Mistakes

- Writing a skill that is too broad, such as `cloud-helper`.
- Writing a skill that is too narrow, such as one command for one exact ticket.
- Putting every possible edge case into `SKILL.md`.
- Hiding important gotchas in a reference file without a clear trigger.
- Using a vague description, causing the skill to trigger at the wrong time.
- Providing a menu of tools instead of a default tool.
- Creating extra files like `README.md` or `QUICKSTART.md` inside the skill folder.

## Recommended Writing Process

1. Start from one real task.
2. Extract the successful workflow.
3. Add concrete gotchas from mistakes or corrections.
4. Add examples only where they clarify behavior.
5. Move long details into `references/`.
6. Add scripts for deterministic repeated work.
7. Test the skill on a similar but not identical task.
8. Revise based on the agent's execution trace, not only the final answer.
