# HCT Shared Claude Code Skills

Shared Claude Code plugins for the Human Centered Tech team.

The goal of this repo is to host a Cloud Code plugin, which can enable us to share the same set of organizational skills. We will no doubt have personal ways of doing things which we record in our own personal skills folders, as well as per project skills files. But if there are things which we are finding we want our agents to do across the board for the whole organization, or if we just want to share knowledge that we think would be helpful for other people to add to their agent behavior, this is the place for it. In general, our goal is to grow rapidly in practical wisdom about the art of software development and of facilitating excellent workflows for businesses.

## Installation

```
/plugin marketplace add Human-Centered-Tech/claude-code-plugins
/plugin install hct-shared-skills@hct-marketplace
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `cross-platform` | Audit and fix a Node.js project for cross-platform compatibility (Windows + macOS + Linux) |

## Contributing

1. Clone this repo
2. Add or edit skills in `skills/<name>/SKILL.md`
3. Bump the version in `.claude-plugin/plugin.json`
4. Push to `main`
