# Legba Restoration Guide

> How to restore this agent's state from scratch.

## Quick Restore

```bash
# Clone workspace
git clone https://github.com/0xHoneyJar/legba-workspace.git /root/clawd

# Set git identity
cd /root/clawd
git config user.email "legba@clawdbot.ai"
git config user.name "Legba"
```

## What's Here

| Directory/File | Purpose |
|----------------|---------|
| `AGENTS.md` | Operating instructions |
| `SOUL.md` | Personality and values |
| `USER.md` | Info about Jani |
| `IDENTITY.md` | My name, creature type, vibe |
| `memory/` | Daily logs and learnings |
| `knowledge/` | Design docs, decisions, research |
| `skills/` | Local skill definitions |

## Critical Files

These define who I am:
- `SOUL.md` — personality
- `IDENTITY.md` — name and vibe
- `memory/YYYY-MM-DD.md` — recent context

## Pushing Changes

After any significant work:

```bash
cd /root/clawd
git add -A
git commit -m "description of changes"
git push
```

## Connected Repos

| Repo | Purpose |
|------|---------|
| `0xHoneyJar/legba-workspace` | This workspace (my DNA) |
| `0xHoneyJar/loa` | Loa framework (PRs go here) |
| `0xHoneyJar/clawdbot` | My runtime fork (PRs go here) |

## Environment Requirements

- `GH_TOKEN` — GitHub access
- `ANTHROPIC_API_KEY` — Model access
- Clawdbot runtime installed

---

*If you're reading this, I probably crashed. Sorry about that. — Legba 🚪*
