# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a personal Claude Code plugin marketplace template - a minimal framework for developing and organizing your own Claude Code plugins.

### Repository Type
- Claude Code plugin marketplace template
- Personal plugin development environment
- Educational starting point for plugin architecture

### Current Plugins
- **example-plugin** - Minimal example plugin demonstrating basic plugin structure with one agent, one command, one skill, and one MCP configuration

## Architecture Overview

### Plugin System Structure

The repository uses a simple three-tier architecture:

1. **Marketplace Layer** (`.claude-plugin/marketplace.json`)
   - Registry of all available plugins
   - Maps plugin names to source directories
   - Metadata for plugin discovery

2. **Plugin Layer** (Individual plugin directories with `.claude-plugin/plugin.json`)
   - Plugin metadata (name, version, author)
   - Commands directory (Markdown files with YAML frontmatter)
   - Skills directory (reusable skill definitions)
   - Agents directory (specialized agent prompts)
   - Optional `.mcp.json` for MCP server integration

3. **Execution Layer** (Commands, Skills, Agents)
   - Commands: User-facing slash commands (e.g., `/hello`)
   - Skills: Reusable patterns and expertise
   - Agents: Specialized autonomous agents

### Example Plugin

The example plugin demonstrates the minimal structure:

- **hello-agent**: Basic agent showing structure and YAML frontmatter
- **/hello**: Simple command demonstrating how commands work
- **code_style**: Basic skill with coding guidelines
- **.mcp.json**: Template MCP server configuration

## Development Workflows

### Adding a New Plugin

```bash
# 1. Copy the example plugin as a template
cp -r example my-plugin

# 2. Update plugin metadata
# Edit both my-plugin/plugin.json and my-plugin/.claude-plugin/plugin.json
# Change name, description, version, and author

# 3. Register in marketplace
# Edit .claude-plugin/marketplace.json and add to plugins array:
# {
#   "name": "my-plugin",
#   "source": "./my-plugin",
#   "description": "Your plugin description"
# }
```

### Adding a Command

Commands are Markdown files with YAML frontmatter:

```bash
# Create command file (filename becomes command name)
# e.g., my-plugin/commands/greet.md becomes /greet
```

**Command file structure**:
```markdown
---
description: Brief description of what this command does
---

# Command Documentation

Instructions for Claude on how to respond when this command is invoked...
```

### Adding an Agent

Agents are specialized prompts for autonomous task execution:

```bash
# Create agent file
# e.g., my-plugin/agents/my-agent.md
```

**Agent file structure**:
```markdown
---
name: agent-name
description: When to use this agent with examples
model: sonnet
color: blue
---

# Agent Prompt

Detailed agent instructions with:
- Core competencies
- How to use this agent
- Examples with <example> tags
- Implementation guidelines
```

### Adding a Skill

Skills are reusable expertise patterns:

```bash
# Create skill file
# e.g., my-plugin/skills/my-skill.md
```

**Skill file structure**:
```markdown
---
name: Skill Name
description: When to use this skill and what it covers
---

# Skill Documentation

Patterns, examples, guidelines, and reference material...
```

### Adding MCP Integration

MCP servers enable integration with external services:

```bash
# Create or update my-plugin/.mcp.json
```

**MCP configuration structure**:
```json
{
  "mcpServers": {
    "ServerName": {
      "command": "npx|docker",
      "args": ["..."],
      "env": {
        "ENV_VAR": "${SYSTEM_ENV_VAR}"
      }
    }
  }
}
```

**Important**: Environment variables use system-level references (`${VAR_NAME}`), never hardcoded values.

## Important Patterns & Conventions

### File Naming
- Commands: lowercase with hyphens (`hello.md` → `/hello`)
- Agents: descriptive names (`hello-agent.md`)
- Skills: underscore-separated (`code_style.md`)
- Configuration: standardized names (`plugin.json`, `marketplace.json`, `.mcp.json`)

### Plugin Metadata Standards
- **Versioning**: Semantic versioning (MAJOR.MINOR.PATCH)
- **Descriptions**: User-friendly, concise explanations
- **Author format**: `{ "name": "Your Name" }`

### Agent Design Patterns

1. **YAML Frontmatter**:
   - `name`: Agent identifier
   - `description`: Triggers and usage examples
   - `model`: Claude model tier (sonnet, opus, haiku)
   - `color`: UI identification color

2. **Prompt Structure**:
   - Core competencies (what the agent knows)
   - How to use this agent
   - Examples with `<example>` tags
   - Implementation guidelines

3. **Example Format**:
```markdown
<example>
Context: When this situation occurs
user: "User's request"
assistant: "How agent responds"
<commentary>
Why this approach is correct
</commentary>
</example>
```

## Common Tasks

### Verify Marketplace Configuration
```bash
cat .claude-plugin/marketplace.json
```

### Check Plugin Metadata
```bash
cat example/.claude-plugin/plugin.json
```

### View MCP Server Configuration
```bash
cat example/.mcp.json
```

### List Available Components
```bash
ls -la example/agents/
ls -la example/commands/
ls -la example/skills/
```

### Update Plugin Version
Edit the plugin's `plugin.json` and increment version number following semantic versioning.

## Plugin Architecture Reference

### How Plugins Are Loaded

1. **Discovery**: Claude Code reads `.claude-plugin/marketplace.json`
2. **Registration**: Each plugin's `plugin.json` is loaded from its source directory
3. **Resource Loading**:
   - Commands from `commands/` directory
   - Agents from `agents/` directory
   - Skills from `skills/` directory
   - MCP servers from `.mcp.json` (if present)
4. **Execution**: Commands become callable via `/command-name` syntax

### Agent vs Skill vs Command

- **Command**: User-facing entry point (invoked via `/command-name`)
- **Skill**: Reusable expertise pattern (referenced in prompts, not directly callable)
- **Agent**: Autonomous specialized worker (invoked via Task tool for complex multi-step work)

**When to use each**:
- Command: User needs to trigger a specific workflow
- Skill: Claude needs reference material for a specific pattern
- Agent: Complex task requiring specialized expertise and multiple steps

## Security Considerations

### Credentials Management
- **NEVER** commit credentials to repository
- Use system environment variables for all sensitive data
- Reference env vars in configuration: `${VAR_NAME}`
- Rotate tokens regularly
- Use minimal required permissions for service accounts

### MCP Server Security
- Authentication handled by MCP protocol
- No credential storage in plugin files
- Use environment variables for all secrets

## Key Files Reference

| File | Purpose |
|------|---------|
| `.claude-plugin/marketplace.json` | Plugin registry |
| `example/.claude-plugin/plugin.json` | Plugin metadata |
| `example/plugin.json` | Duplicate plugin metadata |
| `example/.mcp.json` | MCP server configurations (optional) |
| `example/agents/*.md` | Agent prompts |
| `example/skills/*.md` | Skill patterns |
| `example/commands/*.md` | User-facing commands |

## Testing and Verification

### Test a Command
```bash
claude code
# Then in session:
/hello
```

### Verify Agent Loading
Agents are loaded automatically. Invoke with Task tool or check by asking:
```
What agents are available in this session?
```

### Verify MCP Connection
```bash
# In Claude Code session, test MCP server availability
# (Requires actual MCP server configuration)
```

## Troubleshooting

### Plugin Not Loading
1. Verify plugin is registered in `.claude-plugin/marketplace.json`
2. Check `plugin.json` exists in plugin directory
3. Validate JSON syntax in all configuration files
4. Restart Claude Code session

### Command Not Found
1. Verify command file exists in `commands/` directory
2. Check YAML frontmatter is valid
3. Ensure plugin is properly registered
4. Restart Claude Code session

### Agent Not Working
1. Verify agent file exists in `agents/` directory
2. Check YAML frontmatter includes required fields
3. Ensure agent description includes clear usage examples

## Next Steps

1. **Customize the example plugin** - Update with your own content
2. **Create additional plugins** - Copy example as template
3. **Add MCP integrations** - Connect to external services
4. **Build workflows** - Combine commands, agents, and skills

This is a starting point - customize and expand based on your needs!
