---
tags:
  - folder-structure
---
```text
```text
my-project/ 
	├── .claude/ 
	│     └── settings.json
	│     └── settings.local.json 
	│     └── commands/ 
	│         └── review.md
	│         └── fix-issue.md
	│         └── deploy.md
	│     └── rules/ 
	│         └── code-style.md
	│         └── testing.md
	│         └── api-conventions.md
	│     └── skills/      # Reusable behavior packages 
	│         ├── database-handler/ 
	│                 └── SKILL.md # Skill instructions, name, description 
	│         └── python-testing/ 
	│                 └── SKILL.md 
	│     └── agents/
	│         └── code-reviewer.md
	│         └── security-auditor.md  
	├── CLAUDE.md       # Mandatory: Repo-level instructions
	├── CLAUDE.local.md
	├── README.md # Human-readable documentation 
	└── src/
```