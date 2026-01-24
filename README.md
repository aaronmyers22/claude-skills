# Claude Code Skills

Custom skills for [Claude Code](https://claude.ai/claude-code) CLI.

## What are Skills?

Skills are reusable prompts and patterns that extend Claude Code's capabilities. They provide domain-specific knowledge and can be invoked with `/skillname` in your Claude Code sessions.

## Available Skills

### `/fullstack`

Full-stack T3 development assistant for Next.js applications.

**Features:**
- Project scaffolding with modern architecture
- Component generation with glass UI design system
- tRPC API routes with role-based access
- Prisma database schemas
- Code review for T3 best practices

**Usage:**
```bash
/fullstack scaffold auth        # Add authentication with roles
/fullstack component Button     # Generate a UI component
/fullstack api posts crud       # Create CRUD router
/fullstack schema User          # Generate Prisma model
/fullstack review               # Review code for best practices
```

**Tech Stack:**
- Next.js 15 with App Router
- TypeScript (strict mode)
- tRPC with role-based procedures
- Prisma ORM
- Tailwind CSS with glass design system
- NextAuth.js with credentials provider

## Installation

Skills are stored in `~/.claude/skills/`. Each skill is a directory containing:

- `SKILL.md` - Main skill definition with metadata and instructions
- Additional `.md` files - Patterns, templates, and reference docs

To use these skills:

1. Clone this repo to your skills directory:
   ```bash
   git clone https://github.com/aaronmyers22/claude-skills.git ~/.claude/skills
   ```

2. Or copy individual skill directories:
   ```bash
   cp -r fullstack ~/.claude/skills/
   ```

## Creating Custom Skills

### Skill Structure

```
skillname/
├── SKILL.md        # Required: Main skill file with frontmatter
├── patterns.md     # Optional: Code patterns and best practices
└── templates.md    # Optional: Component/code templates
```

### SKILL.md Format

```markdown
---
name: skillname
description: Brief description shown in skill list
argument-hint: [optional] [arguments]
---

# Skill Title

Instructions for Claude when this skill is invoked...
```

## License

MIT
