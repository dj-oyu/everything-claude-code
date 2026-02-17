---
name: ask-gemini
description: Delegate agentic search and research tasks to Gemini CLI for broader web knowledge and cross-model verification.
---

# Ask Gemini

Use the Gemini CLI as an external agentic search tool. Gemini can browse the web, synthesize information, and return structured answers — complementing Claude's local codebase expertise with broad, up-to-date web knowledge.

## When to Activate

- Need up-to-date information beyond Claude's knowledge cutoff
- Researching unfamiliar libraries, frameworks, or APIs
- Cross-verifying technical decisions with a second model
- Investigating production errors with no obvious local cause
- Gathering context on third-party services, changelogs, or migration guides

## Usage

```bash
gemini -p "your prompt here"
```

The `-p` flag runs Gemini in non-interactive (pipe) mode — it processes the prompt and returns the result directly to stdout.

## Integration Patterns

### Direct research query

```bash
gemini -p "What breaking changes were introduced in Next.js 15?"
```

### Scoped technical lookup

```bash
gemini -p "Show the correct way to configure ESLint flat config for TypeScript in 2025"
```

### Error investigation

```bash
gemini -p "Explain this error and common fixes: ENOSPC: System limit for number of file watchers reached"
```

### Architecture comparison

```bash
gemini -p "Compare Drizzle ORM vs Prisma for serverless PostgreSQL. Pros, cons, and cold start impact."
```

## Best Practices

1. **Be specific** — Include versions, frameworks, and constraints in the prompt
2. **Ask for structure** — Request tables, bullet points, or step-by-step formats for cleaner output
3. **Verify locally** — Treat Gemini output as advisory; always validate against local code and docs
4. **Keep prompts focused** — One question per invocation yields better results than multi-part queries
5. **Quote the prompt** — Always wrap the prompt in double quotes to prevent shell interpretation
