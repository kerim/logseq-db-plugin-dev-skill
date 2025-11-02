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

**Key Difference**: DB graphs have dedicated class/tag pages, but plugin API cannot set class property schemas.

```typescript
// Create a class page (v0.2.3+)
const tagPage = await logseq.Editor.createPage(
  'research',
  {},
  {
    class: true
  } as any
)
```

**IMPORTANT LIMITATION**: The `schema` parameter on `createPage()` creates **page-level schemas**, NOT class schemas. Plugins cannot programmatically set the `:logseq.property.class/properties` field due to security restrictions.

### Alternative: Property Entities with Schemas

Instead of class schemas, use property entities:

```typescript
// Step 1: Create property entities with schemas
await logseq.Editor.upsertProperty('year', {
  type: 'number'
})

await logseq.Editor.upsertProperty('title', {
  type: 'string'
})

await logseq.Editor.upsertProperty('tags', {
  type: 'string',
  cardinality: 'many'
})

// Step 2: Use properties on pages with explicit schemas
const page = await logseq.Editor.createPage(
  'Study on XYZ #research',
  {}
)

// Set each property with its schema
await logseq.Editor.upsertBlockProperty(
  page.uuid,
  'year',
  2025,
  { type: 'number' }
)

await logseq.Editor.upsertBlockProperty(
  page.uuid,
  'title',
  'XYZ Study',
  { type: 'string' }
)
```

**Why This Works**:
- Property entities are created with schemas stored at the property level
- `upsertBlockProperty()` with schema parameter sets typed values on pages
- Properties maintain correct types without class schema inheritance
- Fully automatic - no manual setup required

**What Cannot Be Done**:
```typescript
// ❌ FAILS - Plugins cannot set class/properties
await logseq.Editor.upsertBlockProperty(
  classPage.uuid,
  'class/properties',
  ['year', 'title', 'verified']
)
// Error: "Plugins can only upsert its own properties"
```

The `:logseq.property.class/properties` field is a system property that plugins cannot modify. Users must manually add properties to class schemas via the Logseq UI if they want centralized schema management.

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

## Property Management

### Creating Property Entities (v0.2.3+)

**DB Graphs Only**: Properties can be created as standalone entities with schemas.

```typescript
// Create a property entity with schema
const property = await logseq.Editor.upsertProperty('customField', {
  type: 'string',
  cardinality: 'one'
})

// Property is created with namespace
// Result: :plugin.property.my-plugin-id/customfield

// Get property entity
const prop = await logseq.Editor.getProperty('customField')
console.log(prop.id)  // Database ID
console.log(prop.type)  // 'string'
```

**Available Schema Options**:
```typescript
{
  type: 'default' | 'number' | 'date' | 'datetime' | 'checkbox' | 'url' | 'node' | 'json' | 'string',
  cardinality: 'one' | 'many',  // Default: 'one'
  hide?: boolean,
  public?: boolean
}
```

**Benefits**:
- Properties get database IDs
- Schemas stored at property level
- Reusable across pages
- Type validation automatic

### Setting Properties with Schemas

```typescript
// Set a property on a block/page with schema
await logseq.Editor.upsertBlockProperty(
  page.uuid,
  'year',
  2025,
  { type: 'number' }
)

// Multi-value property
await logseq.Editor.upsertBlockProperty(
  page.uuid,
  'tags',
  ['research', 'science'],
  { type: 'string', cardinality: 'many' }
)
```

**When to use `upsertBlockProperty()` with schema**:
- When you need explicit type control per-property
- When property entity doesn't exist yet
- When you want to ensure type even if property schema changes

**When to use `upsertProperty()` first**:
- When you have a set of standard properties
- When you want centralized schema definitions
- When you want to reuse schemas across many pages

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
5. **Cannot set class schemas programmatically** - Use `upsertProperty()` and `upsertBlockProperty()` instead
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
3. **Cannot set class property schemas programmatically** - `:logseq.property.class/properties` is a system property that plugins cannot modify. Users must manually configure class schemas in the UI.
4. **TypeScript definitions incomplete** - `schema` and `class` parameters not defined

### Class Schema Workaround

Since plugins cannot set class schemas, use this pattern:

```typescript
// Create property entities on plugin init
async function initProperties() {
  const properties = {
    year: { type: 'number' },
    title: { type: 'string' },
    verified: { type: 'checkbox' }
  }

  for (const [name, schema] of Object.entries(properties)) {
    await logseq.Editor.upsertProperty(name, schema)
  }
}

// Use explicit schemas when setting properties on pages
async function createPage(data) {
  const page = await logseq.Editor.createPage(
    `${data.title} #mytag`,
    {}
  )

  await logseq.Editor.upsertBlockProperty(
    page.uuid, 'year', data.year, { type: 'number' }
  )
  await logseq.Editor.upsertBlockProperty(
    page.uuid, 'title', data.title, { type: 'string' }
  )
}
```

This provides typed properties without requiring class schema setup.

## Requirements

- **Logseq**: 0.11.0+ (for DB graphs)
- **@logseq/libs**: 0.2.3+
- **Build tool**: vite-plugin-logseq recommended
- **Testing**: Must test on actual DB graphs, not MD graphs

## Best Practices

1. **Create property entities first** - Use `upsertProperty()` to define reusable properties with schemas
2. **Use explicit schemas** - Pass schema parameter to `upsertBlockProperty()` for type safety
3. **Alternative names for reserved properties** - Avoid `created`, `modified`, `updated`
4. **Tags in page title** - Include tags using `#tag` syntax, not as properties
5. **Convert HTML to markdown** - Use Turndown for DB graphs
6. **Query by unique properties** - Use DOI, URL, etc. for duplicate detection, not page titles
7. **Critical properties first** - Put important properties early (validation failures drop subsequent ones)
8. **Expose test functions** - Use `parent.window` for console testing
9. **Type assertions** - Use `as any` for schema parameters until TypeScript defs updated
10. **Document manual setup** - If class schemas are important for your use case, provide user instructions for manual configuration
