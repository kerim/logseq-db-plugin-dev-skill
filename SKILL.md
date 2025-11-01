---
name: logseq-db-plugin-dev
description: Essential knowledge for developing Logseq plugins for DB (database) graphs. Use this skill when creating or debugging Logseq plugins that work with DB graphs. Claude Code only.
---

# Logseq DB Plugin Development Reference

**Target**: @logseq/libs v0.2.3 | Logseq 0.11.0+ (DB graphs)

This guide focuses on **DB (database) graphs**, which differ fundamentally from markdown/file-based graphs. Critical differences are highlighted throughout.

## Environment Setup

### Package.json
```json
{
  "name": "logseq-example-plugin",
  "version": "1.0.0",
  "main": "dist/index.js",
  "logseq": {
    "id": "logseq-example-plugin",
    "title": "Example Plugin",
    "icon": "./icon.svg"
  },
  "dependencies": {
    "@logseq/libs": "^0.2.3"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "vite": "^5.0.0",
    "vite-plugin-logseq": "^1.1.2"
  }
}
```

### Basic Plugin Entry Point
```typescript
import '@logseq/libs'

async function main() {
  console.log('Plugin loaded')

  logseq.Editor.registerSlashCommand('My Command', async () => {
    // Command implementation
  })
}

logseq.ready(main).catch(console.error)
```

## Creating Pages

### MD Graphs (File-based)
```typescript
// Simple page creation
const page = await logseq.Editor.createPage(
  'Page Name',
  {
    property1: 'value1',
    property2: 'value2'
  }
)
```

### DB Graphs (Database)

**Key Difference**: DB graphs support explicit schema definitions and automatic property namespacing.

```typescript
// Page with explicit schema (v0.2.3+)
const page = await logseq.Editor.createPage(
  'Article Title',
  {
    title: 'My Article',
    year: 2025,
    verified: true,
    url: 'https://example.com'
  },
  {
    schema: {
      title: { type: 'string' },
      year: { type: 'number' },
      verified: { type: 'checkbox' },
      url: { type: 'url' }
    },
    redirect: false,
    createFirstBlock: false
  } as any  // TypeScript defs not yet updated
)
```

**Property Types**:
- `string` - Text
- `number` - Numbers
- `checkbox` - Boolean checkboxes
- `url` - URLs (validated)
- `date` - Date values
- `datetime` - Date with time
- `page` - Page references
- `node` - Block references

**Important**:
- Properties are namespaced as `:plugin.property.{plugin-id}/{property-name}`
- Use `as any` for schema parameter (TypeScript definitions lag behind runtime API)
- Schema parameter eliminates DataCloneError from v0.0.17

## Creating Class/Tag Pages

### MD Graphs
Not applicable - tags are just page references.

### DB Graphs

**Key Difference**: DB graphs have dedicated class/tag pages that can define property schemas for all tagged pages.

```typescript
// Create a class page with schema (v0.2.3+)
const tagPage = await logseq.Editor.createPage(
  'research',  // Class name
  {},
  {
    class: true,
    schema: {
      year: { type: 'number' },
      title: { type: 'string' },
      verified: { type: 'checkbox' },
      publicationDate: { type: 'date' }
    }
  } as any
)

// Pages tagged with #research will inherit this schema
const article = await logseq.Editor.createPage(
  'Study on XYZ #research',
  {
    year: 2025,
    title: 'XYZ Study',
    verified: true,
    publicationDate: '2025-01-15'
  }
)
```

**Important**:
- Class pages get auto-generated ident: `:user.class/research-XXXXXXXX`
- Schema must be passed during creation - no API to add it later
- This enables programmatic tag schema setup (major improvement in v0.2.3)

## Adding Tags to Pages

### MD Graphs
```typescript
// Tags can be added as page properties or in content
const page = await logseq.Editor.createPage(
  'My Page',
  { tags: ['tag1', 'tag2'] }
)
```

### DB Graphs

**Key Difference**: Tags must be included in the page title. No explicit `addTag()` API exists.

```typescript
// ✅ CORRECT - Tags in page title
const page = await logseq.Editor.createPage(
  'My Article #research #science',
  { year: 2025 }
)

// ❌ WRONG - No effect in DB graphs
const page = await logseq.Editor.createPage(
  'My Article',
  { tags: ['research', 'science'] }  // Creates custom property, not actual tags
)

// ❌ WRONG - addTag() method doesn't exist
await logseq.Editor.addTag(page.uuid, 'research')  // Error: "Not existed method"
```

## Creating Blocks

### MD Graphs
```typescript
// Simple block insertion
const block = await logseq.Editor.insertBlock(
  parentBlock,
  'Block content',
  { sibling: true }
)
```

### DB Graphs

**Key Difference**: Blocks in DB graphs have strict UUID requirements and different property handling.

```typescript
// Single block with properties
const block = await logseq.Editor.appendBlockInPage(
  'Page Name',
  'Block content',
  {
    properties: {
      customProp: 'value'
    }
  }
)

// Batch blocks (nested structure)
const blocks = await logseq.Editor.insertBatchBlock(
  parentBlock,
  [
    {
      content: 'Parent block',
      children: [
        { content: 'Child 1' },
        { content: 'Child 2' }
      ]
    }
  ],
  { sibling: false }
)
```

**Important**:
- `properties` parameter in `insertBatchBlock` is **not supported** in DB graphs
- Use `insertBlock` with `properties` option instead for individual blocks
- Blocks without UUIDs get auto-generated UUIDs

## Property Handling

### Reserved Property Names

**Both MD and DB**: Avoid these reserved names - they have special validation:
- `created`
- `modified`
- `updated`

**Issue in DB graphs (v0.2.3)**:
- Reserved names don't throw errors anymore
- BUT they silently drop all subsequent properties
- Workaround: Use alternative names

```typescript
// ❌ PROBLEMATIC - Drops 'title' property
const page = await logseq.Editor.createPage(
  'My Page',
  {
    created: '2025-01-15',
    title: 'Will be dropped'  // This gets lost
  }
)

// ✅ SAFE - Use alternative names
const page = await logseq.Editor.createPage(
  'My Page',
  {
    dateCreated: '2025-01-15',
    publicationDate: '2025-01-15',
    title: 'Works correctly'
  }
)
```

### Property Namespacing

**MD Graphs**: Properties stored as-is in frontmatter.

**DB Graphs**: Plugin properties are automatically namespaced.

```typescript
// Plugin creates this property:
{ title: 'My Article' }

// Stored in DB as:
:plugin.property.my-plugin-id/title "My Article"
```

**Important**: This prevents conflicts but means you can't modify Logseq's built-in properties like `:block/tags`.

## Querying Data

### MD Graphs
Query markdown files directly or use Datalog on file-based database.

### DB Graphs

**Key Difference**: Must use Datalog queries against the database.

```typescript
// Find pages by property value
const results = await logseq.DB.q(`
  [:find (pull ?b [*])
   :where
   [?b :plugin.property.my-plugin/doi "${doi}"]]
`)

// Check if page exists
const existing = await logseq.DB.q(`
  [:find (pull ?b [*])
   :where
   [?b :block/original-name ?name]
   [(= ?name "${pageName}")]]
`)
```

**Important**:
- Use `:block/original-name` for case-sensitive page name lookups
- Property queries must use full namespaced property name
- Results are arrays of arrays: `[[{page1}], [{page2}]]`

## Handling HTML Content

### MD Graphs
Can insert HTML directly into markdown.

### DB Graphs

**Key Difference**: HTML must be converted to markdown first.

```typescript
import TurndownService from 'turndown'

const turndownService = new TurndownService({
  headingStyle: 'atx',
  codeBlockStyle: 'fenced'
})

// Convert HTML to markdown
const markdown = turndownService.turndown(htmlContent)

const block = await logseq.Editor.appendBlockInPage(
  pageName,
  markdown
)
```

## Duplicate Detection

### MD Graphs
Check for existing files.

### DB Graphs

**Key Difference**: Query by unique property, not page title.

```typescript
// ❌ UNRELIABLE - Page titles can change
const existing = await logseq.Editor.getPage(pageName)

// ✅ RELIABLE - Query by unique property (DOI, URL, etc.)
const results = await logseq.DB.q(`
  [:find (pull ?b [*])
   :where
   [?b :plugin.property.my-plugin/doi "${doi}"]]
`)

if (results && results.length > 0) {
  console.log('Page already exists')
}
```

## Testing in Browser Console

Plugins run in iframes. To test functions from browser console:

```typescript
// In your plugin code - expose to parent window
if (typeof parent !== 'undefined' && parent.window) {
  // @ts-ignore
  parent.window.myTestFunction = async function() {
    // Test code here
  }
}

// In browser console
await window.myTestFunction()
```

## Common Gotchas

### DB Graphs Specific

1. **Tags must be in page title** - No `addTag()` API
2. **Properties auto-namespaced** - Can't modify `:block/tags` directly
3. **Reserved names drop subsequent properties** - Use alternative names
4. **HTML must be converted to markdown** - Use Turndown
5. **Schema must be set at creation** - No API to add later
6. **TypeScript defs lag behind** - Use `as any` for schema parameters
7. **Query by unique properties** - Not by page titles

### Both MD and DB

1. **React StrictMode causes double execution** - Avoid if possible
2. **Empty properties auto-hidden** - Set to `null` or `""` to preserve
3. **Case-insensitive tag lookup** - `#Tag` and `#tag` are the same

## API Differences Summary

| Feature | MD Graphs | DB Graphs |
|---------|-----------|-----------|
| Page creation | Simple properties | Schema-based with types |
| Property storage | Frontmatter | Namespaced database entities |
| Tags | Property or content | Must be in page title |
| HTML content | Direct insertion | Convert to markdown |
| Duplicate detection | By filename | By unique property query |
| Class pages | Not applicable | Schema inheritance support |
| Block properties | Frontmatter | Database properties |

## Known Limitations (v0.2.3)

1. **No tag insertion API** - Must use `#tag` in title
2. **Reserved date properties partially broken** - Drop subsequent properties
3. **No schema modification API** - Must set during creation
4. **TypeScript definitions incomplete** - `schema` and `class` parameters not defined

## Requirements

- **Logseq**: 0.11.0+ (for DB graphs)
- **@logseq/libs**: 0.2.3+
- **Build tool**: vite-plugin-logseq recommended
- **Testing**: Must test on actual DB graphs, not MD graphs

## Best Practices

1. Always pass schema parameter for clarity and type safety
2. Use alternative names for date properties (avoid `created`, `modified`)
3. Include tags in page title using `#tag` syntax
4. Convert HTML to markdown using Turndown
5. Query by unique properties (DOI, URL) for duplicate detection
6. Put critical properties first (validation failures drop subsequent ones)
7. Expose test functions to parent window for console testing
8. Use `as any` for schema parameters until TypeScript defs are updated
