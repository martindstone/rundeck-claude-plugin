---
name: Project Overview
description: What this repo is and how its skills are structured
type: project
---

This is a Claude Code plugin for Rundeck. It provides two skills:
- `skills/rundeck/SKILL.md` — generates, imports, and executes Rundeck job YAML using the `rd` CLI
- `skills/pagerduty/SKILL.md` — interacts with the PagerDuty REST API

The plugin connects to a live Rundeck instance at `$RD_URL` using `$RD_TOKEN`.
The connected instance has the PagerDuty Plugins bundle (v5.17.0) installed with ~16 WorkflowStep plugins.

**Why:** The SKILL.md files are the guidance the model reads at runtime — improving them reduces round-trips and failures.
**How to apply:** When editing skills, focus on concrete type names, required fields, and explicit "prefer plugin over script" rules.
