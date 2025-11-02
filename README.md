# Logseq DB Plugin Development Skill

A comprehensive Claude skill for developing Logseq plugins that work with DB (database) graphs.

## What is this?

This is a Claude skill that provides essential knowledge for developing Logseq plugins specifically for DB graphs. DB graphs use a fundamentally different architecture from the original markdown-based graphs, requiring different APIs and approaches.

## Key Differences: MD vs DB Graphs

| Feature | MD Graphs (File-based) | DB Graphs (Database) |
|---------|------------------------|----------------------|
| **Page creation** | Simple properties in frontmatter | Schema-based with explicit types |
| **Property storage** | YAML frontmatter | Namespaced database entities (`:plugin.property.{id}/name`) |
| **Tags** | Can use properties or content | Must be in page title (`#tag` syntax) |
| **HTML content** | Direct insertion allowed | Must convert to markdown first |
| **Duplicate detection** | Check by filename | Query by unique property values |
| **Class/tag pages** | Not applicable | Support schema inheritance |
| **Property types** | Inferred from markdown | Explicit schema or type inference |
| **Block properties** | Frontmatter in markdown | Database properties |

### Critical DB-Specific Requirements

1. **Tags must be in page title** - No `addTag()` API exists
2. **Properties auto-namespaced** - Can't modify built-in properties like `:block/tags`
3. **Schema at creation only** - Must pass schema when creating pages, no API to add later
4. **HTML requires conversion** - Use Turndown to convert HTML to markdown
5. **Query by properties** - Use unique properties (DOI, URL) not page titles for duplicates

## What's covered?

- **Environment setup** - Package.json, dependencies, build tools
- **Page creation** - Regular pages and class/tag pages with schemas (v0.2.3+)
- **Property handling** - Type system, namespacing, reserved names
- **Block creation** - Single blocks and batch operations
- **Querying** - Datalog queries for DB graphs
- **Tag management** - How to properly tag pages in DB graphs
- **Common gotchas** - Property validation, StrictMode, reserved names
- **Best practices** - Schema usage, duplicate detection, testing

## Key Features (v0.2.3)

### Schema Support

Both regular and class pages support explicit schemas:

```typescript
// Regular page with schema
const page = await logseq.Editor.createPage(
  'Article Title',
  { year: 2025, title: 'Study' },
  {
    schema: {
      year: { type: 'number' },
      title: { type: 'string' }
    }
  } as any
)

// Class page with schema (enables tag-based property inheritance)
const tagPage = await logseq.Editor.createPage(
  'research',
  {},
  {
    class: true,
    schema: {
      year: { type: 'number' },
      title: { type: 'string' }
    }
  } as any
)
```

### Property Types

- `string` - Text
- `number` - Numbers
- `checkbox` - Booleans
- `url` - URLs (validated)
- `date` / `datetime` - Dates
- `page` - Page references
- `node` - Block references

### Tag Syntax

Tags must be in the page title:

```typescript
// ✅ Correct
const page = await logseq.Editor.createPage(
  'My Article #research #science',
  { year: 2025 }
)

// ❌ Wrong - creates custom property, not a tag
const page = await logseq.Editor.createPage(
  'My Article',
  { tags: ['research'] }
)
```

## Who is this for?

- Plugin developers building for Logseq DB graphs
- Developers migrating markdown-based plugins to DB graphs
- Anyone debugging plugin issues with properties, types, or schemas
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

**Note**: This skill is marked "Claude Code only" and may have limited functionality in Claude Desktop.

1. Download this repository as a ZIP
2. In Claude Desktop: Settings → Skills → Add Skill → Upload ZIP
3. The skill will be available in all conversations

## What makes this different?

This skill is based on:
- Actual testing with @logseq/libs v0.2.3
- Official Logseq source code analysis
- Documented API differences between MD and DB graphs
- Best practices for DB graph development

It provides clear guidance on:
- Schema parameter usage (works in v0.2.3)
- Class page schema creation (works in v0.2.3)
- Property namespacing and limitations
- Reserved property names to avoid
- Proper tag insertion techniques
- HTML to Markdown conversion

## Known Limitations (v0.2.3)

1. **No tag insertion API** - Must use `#tag` in title
2. **Reserved property names** - Drop subsequent properties (`created`, `modified`, `updated`)
3. **No schema modification API** - Must set during page creation
4. **TypeScript definitions incomplete** - `schema` and `class` parameters require `as any`

## Requirements

- **@logseq/libs**: v0.2.3+
- **Logseq**: 0.11.0+ (for DB graphs)
- **Build tool**: vite-plugin-logseq recommended
- **Node.js**: For building plugins

## Contributing

Found an issue or have improvements? Contributions welcome!

1. Test your changes with actual Logseq DB graphs
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

Reflects @logseq/libs v0.2.3 and Logseq 0.11.0+ DB graphs. Will be updated as the API evolves.
