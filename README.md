# AI Tools

Personal repository for AI-assisted development workflow tools and configurations.

## Structure

- `sdlc/` - Software Development Lifecycle tools and workflows
  - `pr-workflow-definition.md` - Master PR workflow definition
  - `tests/` - Validation scripts
  - `examples/` - Example files and templates

## Usage

This repository contains the configuration and workflow definitions for AI-assisted development with Claude Code.

### PR Workflow

See `sdlc/pr-workflow-definition.md` for the complete workflow specification.

Quick start:
```bash
start-workflow
```

## Configuration

Main configuration files:
- `~/.pr-workflow-config` - Workflow preferences (Jira project, Slack channel, etc.)
- `~/.jira-github-slack-helpers.sh` - Shell helper functions
- `~/CLAUDE.md` - Claude Code guidance

## Documentation

- [PR Workflow Definition](sdlc/pr-workflow-definition.md)
- [Planning Questions Reference](~/.pr-planning-questions.md)
- [Workflow Examples](~/.pr-workflow-example.md)

## Contributing

To update the workflow:
1. Edit `sdlc/pr-workflow-definition.md`
2. Update corresponding shell functions in `~/.jira-github-slack-helpers.sh`
3. Update tests if needed
4. Commit and push

## Version

Current version: 1.0.0 (Initial implementation)
