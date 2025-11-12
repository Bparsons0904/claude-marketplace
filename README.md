# Personal Claude Code Plugin Marketplace

A minimal template for creating and organizing your own Claude Code plugins.

## Overview

This repository provides a basic framework for developing Claude Code plugins. It includes a simple example plugin that demonstrates the core plugin architecture.

### What's Included

- **Example Plugin** - A minimal plugin with basic examples of:
  - Agent (hello-agent)
  - Command (/hello)
  - Skill (code_style)
  - MCP server configuration

## Quick Start

### 1. Clone and Customize

```bash
# Clone or copy this repository
git clone <your-repo-url>
cd marketplace

# Update marketplace configuration
# Edit .claude-plugin/marketplace.json with your name
```

### 2. Create Your First Plugin

```bash
# Copy the example plugin as a template
cp -r example my-plugin

# Update plugin metadata
# Edit my-plugin/plugin.json and my-plugin/.claude-plugin/plugin.json
```

### 3. Register Your Plugin

Add your plugin to `.claude-plugin/marketplace.json`:

```json
{
  "plugins": [
    {
      "name": "my-plugin",
      "source": "./my-plugin",
      "description": "Your plugin description"
    }
  ]
}
```

### 4. Test in Claude Code

```bash
# Start Claude Code in this directory
claude code

# Test your command
/hello
```

## Plugin Structure

```
marketplace/
├── .claude-plugin/
│   └── marketplace.json          # Plugin registry
├── example/                       # Example plugin
│   ├── .claude-plugin/
│   │   └── plugin.json           # Plugin metadata
│   ├── agents/
│   │   └── hello-agent.md        # Agent definition
│   ├── commands/
│   │   └── hello.md              # Command definition
│   ├── skills/
│   │   └── code_style.md         # Skill definition
│   ├── .mcp.json                 # MCP server config (optional)
│   └── plugin.json               # Plugin metadata
└── README.md
```

## Plugin Components

### Commands

Commands are user-facing actions invoked via `/command-name`.

**Structure:**
```markdown
---
description: Brief description of what this command does
---

# Command Instructions

Detailed instructions for Claude on how to respond...
```

**File location:** `plugin/commands/command-name.md`

### Agents

Agents are specialized autonomous workers for complex tasks.

**Structure:**
```markdown
---
name: agent-name
description: When to use this agent
model: sonnet
color: blue
---

# Agent Prompt

Core competencies and instructions...
```

**File location:** `plugin/agents/agent-name.md`

### Skills

Skills are reusable patterns and expertise that Claude can reference.

**Structure:**
```markdown
---
name: Skill Name
description: When to use this skill
---

# Skill Content

Guidelines, patterns, and examples...
```

**File location:** `plugin/skills/skill-name.md`

### MCP Servers (Optional)

MCP (Model Context Protocol) servers enable integration with external services.

**Structure:**
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

**File location:** `plugin/.mcp.json`

## Development Workflow

### Adding a Command

1. Create `plugin/commands/my-command.md`
2. Add YAML frontmatter with description
3. Write instructions for Claude
4. Test with `/my-command`

### Adding an Agent

1. Create `plugin/agents/my-agent.md`
2. Add YAML frontmatter (name, description, model, color)
3. Write agent prompt with core competencies
4. Agent is automatically available

### Adding a Skill

1. Create `plugin/skills/my-skill.md`
2. Add YAML frontmatter with name and description
3. Write skill content (patterns, guidelines, examples)
4. Reference in commands or agent prompts

### Adding MCP Integration

1. Create or update `plugin/.mcp.json`
2. Add server configuration with command and args
3. Use environment variables for secrets
4. Restart Claude Code session to load MCP servers

## Best Practices

### Naming Conventions

- **Commands:** lowercase-with-hyphens.md → `/command-name`
- **Agents:** descriptive-name.md
- **Skills:** snake_case.md
- **Plugins:** lowercase-with-hyphens

### Security

- **Never commit credentials** - use environment variables
- Reference env vars: `${VAR_NAME}` in configuration
- Use minimal required permissions for service accounts

### Organization

- Keep plugins focused on a specific domain or workflow
- Group related commands, agents, and skills together
- Use clear, descriptive names for all components

## Example Plugin

The included `example` plugin demonstrates:

- **hello-agent** - Shows basic agent structure with YAML frontmatter
- **/hello command** - Demonstrates simple command with instructions
- **code_style skill** - Shows how to define reusable guidelines
- **.mcp.json** - Template for MCP server configuration

Use these as templates for your own plugins.

## Troubleshooting

### Plugin not loading
- Verify plugin is registered in `.claude-plugin/marketplace.json`
- Check that `plugin.json` exists in both root and `.claude-plugin/` directories
- Validate JSON syntax
- Restart Claude Code

### Command not found
- Verify command file exists in `commands/` directory
- Check YAML frontmatter is valid
- Ensure filename matches command name
- Restart Claude Code

### MCP connection failed
- Verify environment variables are set
- Check Docker is running (if using Docker-based MCP servers)
- Validate network connectivity
- Check MCP server URL and credentials

## Resources

- [Claude Code Documentation](https://docs.claude.com/en/docs/claude-code)
- [MCP Documentation](https://github.com/anthropics/mcp)

## License

Customize this section with your preferred license.
