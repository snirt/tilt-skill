# Tilt Skill for AI Coding Agents

A [Tilt](https://tilt.dev) skill that teaches AI coding agents how to write Tiltfiles, use the Tilt CLI, and debug dev environments.

## What It Covers

- **Tiltfile API** - `docker_build`, `helm_resource`, `local_resource`, `docker_compose`, `dc_resource`, live_update, config system, resource dependencies, trigger modes
- **CLI Discovery** - Teaches the agent to use `tilt --help` and `tilt api-resources` at runtime instead of relying on stale docs
- **Patterns** - Helm + image wiring, Docker Compose overrides, health probes, session persistence, conditional resources
- **Pitfalls** - Common mistakes and their fixes

## Installation

### Claude Code (Superpowers)

```bash
# From your project directory
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/snirt/tilt-skill.git tilt
```

Or add as a git submodule:

```bash
git submodule add https://github.com/snirt/tilt-skill.git .claude/skills/tilt
```

### Global Installation (All Projects)

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/snirt/tilt-skill.git tilt
```

### Manual

Copy `SKILL.md` into your agent's skill directory. The exact path depends on your setup.

## Usage

The skill activates automatically when you work on Tiltfiles or ask about Tilt. You can also invoke it directly:

```
/tilt
```

## Updating

```bash
cd <path-to-skill>/tilt
git pull
```
