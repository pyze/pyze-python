# Pyze Python Plugin

Claude Code plugin providing Python-specific development patterns and companion skills.

## Project Structure
- `skills/` - Skill definitions (SKILL.md files with frontmatter)
- `.claude-plugin/plugin.json` - Plugin manifest

## Companion Plugin
This plugin provides Python-specific examples for general principles in pyze-workflow.

Companion skill mappings:
- decomplection-python → pyze-workflow's decomplection-first-design
- testing-patterns-python → pyze-workflow's testing-patterns
- error-handling-python → pyze-workflow's error-handling-patterns
- caching-and-purity-python → pyze-workflow's caching-and-purity
- bdd-scenarios-python → pyze-workflow's bdd-scenarios
- tool-selection-python → pyze-workflow's tool-selection

## Testing Changes
- Start a new Claude Code session to test skill changes
- Use `/reload-plugins` to reload without restarting

## Contributing
- Semver: major (breaking renames/removals), minor (new skills), patch (content fixes)
