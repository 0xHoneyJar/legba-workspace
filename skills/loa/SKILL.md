---
name: loa
description: Agent-driven development framework that orchestrates the complete product lifecycle using specialized AI agents. Use this skill when users want to plan projects, architect systems, implement sprints, review code, or audit security using the Loa methodology. The Loa framework is installed at /root/loa and can be mounted onto any project.
---

# Loa Framework

Agent-driven development framework installed at `/root/loa`. Orchestrates the complete product lifecycle using 9 specialized AI agents.

## Installation

The Loa framework is pre-installed at `/root/loa`. To use it on a project:

```bash
# Clone or navigate to your project
cd /root/clawd/projects/my-project

# Mount Loa onto the project
/root/loa/.claude/scripts/mount-loa.sh .

# Now you can run Loa commands with Claude Code
claude
```

## Running Loa Commands

Once Loa is mounted, use Claude Code to run commands:

```bash
# Start Claude Code in the project directory
cd /root/clawd/projects/my-project
claude

# Then use Loa commands:
# /plan-and-analyze - Start PRD discovery
# /architect - Create software design
# /sprint-plan - Break into tasks
# /implement sprint-1 - Build code
```

## Core Workflow (Sequential Phases)

| Phase | Command | Agent Role | Output |
|-------|---------|------------|--------|
| 1 | `/plan-and-analyze` | Product Manager | `grimoires/loa/prd.md` |
| 2 | `/architect` | Software Architect | `grimoires/loa/sdd.md` |
| 3 | `/sprint-plan` | Technical PM | `grimoires/loa/sprint.md` |
| 4 | `/implement sprint-N` | Senior Engineer | Code + reviewer.md |
| 5 | `/review-sprint sprint-N` | Tech Lead | engineer-feedback.md |
| 5.5 | `/audit-sprint sprint-N` | Security Auditor | auditor-sprint-feedback.md |
| 6 | `/deploy-production` | DevOps Architect | Infrastructure |

## Key Commands

### Discovery & Planning
- `/plan-and-analyze [--fresh]` - PRD discovery with codebase grounding
- `/architect [background]` - Software Design Document
- `/sprint-plan [background]` - Task breakdown with acceptance criteria

### Implementation
- `/implement sprint-N` - Execute sprint tasks
- `/review-sprint sprint-N` - Code review quality gate
- `/audit-sprint sprint-N` - Security audit final gate

### Autonomous Mode
- `/run sprint-N` - Autonomous sprint execution
- `/run sprint-plan` - Execute all sprints
- `/run-status` / `/run-halt` / `/run-resume` - Control autonomous mode

### Codebase Analysis
- `/ride` - Analyze existing codebase, generate artifacts
- `/reality [query]` - Quick codebase reality check

### Validation
- `/validate architecture` - SDD compliance
- `/validate security` - OWASP Top 10 scan
- `/validate tests` - Test quality
- `/validate goals` - PRD goal achievement

## Project Setup Example

```bash
# 1. Create project directory
mkdir -p /root/clawd/projects/my-app
cd /root/clawd/projects/my-app

# 2. Initialize git repo
git init

# 3. Mount Loa
/root/loa/.claude/scripts/mount-loa.sh .

# 4. Start Claude Code and begin
claude
# Then: /plan-and-analyze
```

## Three-Zone Model

| Zone | Path | Owner |
|------|------|-------|
| System | `.claude/` | Framework (never edit) |
| State | `grimoires/`, `.beads/` | Project |
| App | `src/`, `lib/` | Developer |

## Framework Location

- **Loa repo**: `/root/loa`
- **Mount script**: `/root/loa/.claude/scripts/mount-loa.sh`
- **Update script**: `/root/loa/.claude/scripts/update.sh`

## Environment

Claude Code requires `ANTHROPIC_API_KEY` which is already configured in this container.
# Force rebuild Fri 30 Jan 2026 17:50:08 AEDT
