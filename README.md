# skillful

A Claude Code marketplace with a suite of reusable skills for use across multiple projects.

## Skills

| Skill | Description |
|-------|-------------|
| `code-review` | Reviews the changes for one ticket in a fast-moving, legacy or vibe-coded codebase: code hygiene, comments, acceptance criteria, error handling, bugs and tests. |

## Structure

```
.claude-plugin/
  marketplace.json        # Marketplace catalog
plugins/
  skillful/
    .claude-plugin/
      plugin.json         # Plugin manifest
    skills/
      <name>/SKILL.md     # One folder per skill
```

## Installation

From GitHub:

```
/plugin marketplace add cesRay/skillful
/plugin install skillful@skillful
```

From a local copy:

```
/plugin marketplace add /path/to/skillful
/plugin install skillful@skillful
```

## Add a skill

1. Create `plugins/skillful/skills/<name>/SKILL.md` with `name` and `description` in the frontmatter.
2. Bump the version in `plugins/skillful/.claude-plugin/plugin.json`.
3. Run `claude plugin validate .` before you commit.

## Add another plugin

Create `plugins/<plugin>/` with its own `.claude-plugin/plugin.json`, and register it in `.claude-plugin/marketplace.json`.
