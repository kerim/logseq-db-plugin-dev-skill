# Logseq DB Plugin Development Skill

A comprehensive Claude skill for developing Logseq plugins that work with DB (database) graphs.

## What is this?

This is a Claude skill that provides essential knowledge for developing Logseq plugins specifically for DB graphs. DB graphs use a fundamentally different architecture from the original markdown-based graphs, requiring different APIs and approaches.

## What's covered?

- **Basic plugin structure** - Package.json, entry points, slash commands, toolbar buttons
- **DB-specific APIs** - Creating pages with typed properties, blocks, tags, and nested structures
- **Type inference** - How Logseq automatically infers property types from JavaScript values
- **React integration** - Complete setup guide for React UIs with proper bundling
- **Common pitfalls** - Schema parameter issues, StrictMode problems, duplicate operations
- **Best practices** - Error handling, notifications, debugging, duplicate detection
- **Working examples** - Copy-paste ready code with explanations

## Key insights

### Property Type Inference (Critical!)

The documented `schema` parameter doesn't work in @logseq/libs v0.0.17. Instead, Logseq infers types from JavaScript values:

```typescript
const page = await logseq.Editor.createPage('My Page', {
  title: 'Text',           // string → text type
  year: 2025,              // number → number type
  verified: true,          // boolean → checkbox type
  url: 'https://...',      // string → url type (auto-detected)
  tags: ['a', 'b']         // array → multi-value set
})
```

### React Integration (Critical!)

Logseq plugins with React require specific setup:

- **Must** use `index.html` entry point, not just JS
- **Must** use `vite-plugin-logseq` for bundling
- **Do not** use React.StrictMode (causes duplicate function calls)
- Create React root **once** at plugin load, render on demand
- Use global locks for async operations to prevent duplicates

### Tag Syntax (Critical!)

Tags must be in the page title using `#tag` syntax:

```typescript
// ✅ Correct
const page = await logseq.Editor.createPage('My Article #research #science', {...})

// ❌ Wrong - creates custom property, not a tag
const page = await logseq.Editor.createPage('My Article', { tags: ['research'] })
```

## Who is this for?

- Plugin developers building for Logseq DB graphs
- Developers migrating markdown-based plugins to DB graphs
- Anyone debugging plugin issues with properties, types, or React UIs
- Claude Code users who want plugin development assistance

## How to use

### In Claude Code (recommended)

1. Install to your skills directory:
```bash
cd ~/.claude/skills
git clone https://github.com/kerim/logseq-db-plugin-dev-skill.git logseq-db-plugin-dev
```

2. Use in your prompts:
```
Use the logseq-db-plugin-dev skill to help me create a Zotero import plugin
```

Claude Code will automatically load the skill and provide expert guidance.

### In Claude Desktop

1. Download this repository as a ZIP
2. In Claude Desktop: Settings → Skills → Add Skill → Upload ZIP
3. The skill will be available in all conversations

## What makes this different?

This skill is based on:
- Actual testing and debugging of real plugins
- Official Logseq source code analysis
- Documented workarounds for known issues
- Best practices discovered through trial and error

It includes solutions to common problems that aren't well-documented:
- Why the `schema` parameter causes `DataCloneError`
- Why React.StrictMode breaks plugins
- How to prevent duplicate operations
- How to properly tag pages programmatically
- How to convert HTML to Markdown for abstracts

## Requirements

- **@logseq/libs**: v0.0.17
- **Logseq**: 0.10.0+ (for DB graphs)
- **Node.js**: For building plugins
- **pnpm/npm**: Package manager

## Contributing

Found an issue or have improvements? Contributions welcome!

1. Test your changes with actual Logseq plugins
2. Document what you tested and the results
3. Submit a PR with clear explanations

## License

MIT

## Related Resources

- [Official Plugin API](https://logseq.github.io/plugins/)
- [Plugin Samples](https://github.com/logseq/logseq-plugin-samples)
- [vite-plugin-logseq](https://github.com/pengx17/logseq-plugin-template-react)
- [Community Plugins](https://github.com/logseq/marketplace)

## Status

DB graphs are in beta. This skill reflects the current state of @logseq/libs v0.0.17 and will be updated as the API evolves.
