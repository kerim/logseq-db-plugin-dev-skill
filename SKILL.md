---
name: logseq-db-plugin-dev
description: Essential knowledge for developing Logseq plugins for DB (database) graphs. Use this skill when creating or debugging Logseq plugins that work with DB graphs. Claude Code only.
---

# Logseq DB Plugin Development

**IMPORTANT**: This skill is for **Logseq DB (database) graphs only**, not markdown/file-based graphs. DB graphs use a fundamentally different architecture.

## Overview

Logseq DB graphs are database-backed knowledge graphs that differ significantly from the original markdown-based graphs. Plugins must be specifically designed for DB graphs using the appropriate APIs.

## Basic Plugin Structure

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
    "@logseq/libs": "^0.0.17"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "vite": "^5.0.0"
  }
}
```

### Basic Plugin Entry Point (src/index.ts)
```typescript
import '@logseq/libs'

async function main() {
  console.log('Plugin loaded')

  // Register slash command
  logseq.Editor.registerSlashCommand('Example Command', async () => {
    // Command implementation
  })

  // Add toolbar button (optional)
  logseq.provideModel({
    async exampleAction() {
      // Action implementation
    }
  })

  logseq.App.registerUIItem('toolbar', {
    key: 'example-plugin',
    template: `
      <a data-on-click="exampleAction"
         class="button"
         title="Example action">
        <span>🔧</span>
      </a>
    `
  })
}

logseq.ready(main).catch(console.error)
```

## MD Plugin API Reference (Legacy - File-based Graphs)

These APIs work for **markdown/file-based graphs**. DB graphs require different approaches (see DB-specific sections below).

### Creating Pages (MD)
```typescript
// Create page with properties (MD graphs)
const page = await logseq.Editor.createPage(
  pageName,
  {
    property1: 'value1',
    property2: 'value2'
  },
  {
    redirect: false,
    createFirstBlock: false
  }
)
```

### Creating Blocks (MD)
```typescript
// Append block to page
const block = await logseq.Editor.appendBlockInPage(
  pageName,
  'Block content',
  {
    properties: {
      key: 'value'
    }
  }
)

// Insert nested blocks
await logseq.Editor.insertBatchBlock(
  parentBlockUUID,
  [
    {
      content: 'First block',
      children: [
        { content: 'Nested block', children: [] }
      ]
    },
    {
      content: 'Second block',
      children: []
    }
  ]
)
```

### Setting Properties (MD)
```typescript
// Set property on block/page
await logseq.Editor.upsertBlockProperty(
  blockUUID,
  'propertyName',
  'value'
)

// Remove property
await logseq.Editor.removeBlockProperty(
  blockUUID,
  'propertyName'
)

// Get property
const value = await logseq.Editor.getBlockProperty(
  blockUUID,
  'propertyName'
)

// Get all properties
const props = await logseq.Editor.getBlockProperties(blockUUID)
```

## DB-Specific APIs (Database Graphs)

**CRITICAL**: DB graphs require a fundamentally different approach to creating pages with properties. You MUST pass properties AND schema together at creation time.

### Key Differences from MD Graphs

1. **Pages are Nodes**: In DB graphs, both pages and blocks are nodes with similar behavior
2. **Typed Properties**: DB graphs support typed properties (text, number, date, datetime, checkbox, url, node, json)
3. **Schema Required**: When creating pages/blocks with properties, you MUST provide a schema to define property types
4. **Tags as Classes**: Tags created with `#NAME` syntax can have properties defined on them
5. **Property Auto-Hide**: Empty properties automatically hide in DB graphs
6. **Template Auto-Apply**: Templates can auto-apply to pages with specific tags

### Creating Pages with Properties (DB)

**VERIFIED WORKING SYNTAX** (tested and confirmed):

```typescript
// Create page with typed properties - CORRECT WAY FOR DB GRAPHS
// DO NOT include schema parameter - it causes serialization errors
const page = await logseq.Editor.createPage(
  'Page Name',
  {
    // Properties (values) - types are inferred from JavaScript values
    title: 'Test Article',           // String
    year: 2025,                       // Number (use actual number, not string)
    verified: true,                   // Boolean (use actual boolean)
    link: 'https://example.com',      // String (will be recognized as URL)
    categories: ['cat1', 'cat2'],     // Array becomes multi-value set
    tags: ['tag1', 'tag2'],           // Array becomes multi-value set
    author: 'Jane Doe'                // String
  },
  {
    redirect: false,
    createFirstBlock: false
  }
)
```

**Critical Findings**:
- ✅ **DO NOT use `schema` parameter** - causes `DataCloneError` (function serialization issue)
- ✅ **Types are auto-inferred** from JavaScript values:
  - Number: Use actual number `2025` not string `"2025"`
  - Boolean: Use actual boolean `true` not string `"true"`
  - URL: Strings with URLs are recognized automatically
  - Multi-value: Arrays become sets `["a", "b"]` → `#{"a" "b"}`
- ✅ **Properties are namespaced**: `:plugin.property.{plugin-name}/{property-name}`
- ✅ **Empty strings auto-hide** in the UI
- ⚠️  **Date properties**:
  - Certain property names (e.g., `created`, `modified`) are auto-detected as date types
  - Error: `"should be a journal date"` - expects page reference to journal page, not timestamp or string
  - Both `"YYYY-MM-DD"` strings and Unix timestamps fail validation
  - **Workaround**: Use different property names (e.g., `dateCreated`, `publicationDate`) or use text type
  - **Critical**: When a property fails validation, ALL subsequent properties are silently dropped
- ⚠️  **Page references**: Unknown if possible without schema parameter

### Tagging Pages (DB)

**CRITICAL**: To add tags to a page in DB graphs, include `#tagname` syntax in the page title. There is NO explicit `addTag()` API.

```typescript
// ✅ CORRECT - Tag in page title
const page = await logseq.Editor.createPage(
  'My Article #research #science',  // Tags parsed from title
  {
    title: 'Article Title',
    year: 2025
  },
  {
    redirect: false,
    createFirstBlock: false
  }
)
// Result: :block/tags #{"Page" "research" "science"}

// ❌ WRONG - Custom property approach
const page = await logseq.Editor.createPage(
  'My Article',
  {
    tags: ['research', 'science']  // Creates :plugin.property.{id}/tags, NOT :block/tags
  }
)

// ❌ WRONG - upsertBlockProperty approach
await logseq.Editor.upsertBlockProperty(page.uuid, 'tags', ['research'])
// Still creates :plugin.property.{id}/tags, NOT :block/tags
// Also throws "Invalid datascript entities" error

// ❌ WRONG - Tag in block content (tags the block, not the page)
await logseq.Editor.appendBlockInPage('My Article', '#research')
// Result: Block has tag, but page's :block/tags only has "Page"
```

**Key Points**:
- Tags must be in page title using `#tagname` syntax
- The `#tag` portion is parsed and removed from the displayed title
- Tags are stored in `:block/tags` property (not a custom property)
- Every page automatically has the `Page` tag
- Custom `tags` properties are NOT the same as `:block/tags`

### Creating Tag/Class Pages (DB)

**VERIFIED WORKING** (tested in POC 4): Tag/class pages can be created programmatically.

```typescript
// Create a tag (class) page
const tagPage = await logseq.Editor.createPage(
  'Tag Name',
  {},
  // @ts-ignore - class option exists but not in type definitions
  { class: true }
)
// Creates :user.class/tag-name-XXXXXXXX
```

**Important Notes:**
- Tag/class pages are created with an auto-generated ident (`:user.class/tag-name-XXXXXXXX`)
- These are distinct from regular pages - they can have property schemas attached
- The `class: true` option is not in TypeScript definitions, use `@ts-ignore`

### Property Schemas on Tags (DB) - UI-ONLY ⚠️

**CRITICAL FINDING** (POC 4): Property schemas CANNOT be defined programmatically via plugin API.

**How Logseq stores schemas internally:**
```clojure
; Class/tag with schema (UI-created)
:user.class/my-tag
{:logseq.property.class/properties [{:db/id 232} {:db/id 233}]}  ; References to property entities

; Property definition entity
:user.property/year
{:logseq.property/type :number  ; Keyword type, not string!
 :db/id 232}
```

**Why plugins can't set schemas:**
1. `:logseq.property.class/properties` requires **db/id references** to property definition entities
2. Plugin API restricts: "Plugins can only upsert its own properties"
3. No API exists to create property definition entities
4. Property types are **keywords** (`:number`, `:default`), not strings (`"number"`, `"text"`)

**What doesn't work:**
```typescript
// ❌ All these approaches FAIL
await logseq.Editor.upsertBlockProperty(tagPage.uuid, 'logseq.property/schema', {...})
// Error: "Plugins can only upsert its own properties"

await logseq.Editor.upsertBlockProperty(tagPage.uuid, 'year', 'number')
// Creates custom property VALUE "number", not a schema

await logseq.Editor.upsertBlockProperty(tagPage.uuid, 'year', { type: 'number' })
// Creates custom property, not a schema
```

**Workaround: Use Type Inference (Recommended)**

Since programmatic schema definition isn't possible, use JavaScript value types for automatic type inference:

```typescript
// ✅ RECOMMENDED - Types inferred from JS values
const page = await logseq.Editor.createPage(
  'My Article #zot',  // Include tag in title
  {
    title: 'Article Title',          // string → text type
    year: 2025,                       // number → number type
    verified: true,                   // boolean → checkbox type
    url: 'https://example.com',       // string → url type (auto-detected)
    authors: ['Author 1', 'Author 2'] // array → multi-value text
  },
  {
    redirect: false,
    createFirstBlock: false
  }
)
```

**Manual Schema Setup (Optional):**
Users CAN manually create tag/class pages in the Logseq UI and define property schemas, which will apply to all pages with that tag. But plugins cannot do this programmatically.

### Setting Properties on Existing Pages/Blocks

After page creation, you can still use `upsertBlockProperty`:

```typescript
// Set individual property (type inferred from value)
await logseq.Editor.upsertBlockProperty(page.uuid, 'title', 'New Title')
await logseq.Editor.upsertBlockProperty(page.uuid, 'count', 42)
await logseq.Editor.upsertBlockProperty(page.uuid, 'active', true)

// Set complex property (JSON)
await logseq.Editor.upsertBlockProperty(page.uuid, 'metadata', {
  a: 1,
  b: [2, 3]
})
```

### Creating Blocks with Properties (DB)

```typescript
// Insert block with properties and schema
const block = await logseq.Editor.insertBlock(
  parentPage,
  'Block content',
  {
    properties: {
      checked: true,
      link: 'https://logseq.com',
      count: 42
    },
    schema: {
      checked: { type: 'checkbox' },
      link: { type: 'url' },
      count: { type: 'number' }
    }
  }
)

// Insert batch blocks with properties
await logseq.Editor.insertBatchBlock(
  pageUUID,
  [
    {
      content: 'First block',
      properties: {
        related: 'Page 1',
        tags: ['Page 2', 'Page 3']
      }
    }
  ],
  {
    schema: {
      related: { type: 'page' },
      tags: { type: 'page' }
    }
  }
)
```

### Creating Nested Block Structures (DB)

**VERIFIED WORKING** (tested in POC 3): `insertBatchBlock` works perfectly for creating nested block hierarchies.

```typescript
// ✅ WORKS PERFECTLY - Nested blocks with children array
await logseq.Editor.insertBatchBlock(parentBlockUUID, [
  {
    content: 'Level 1 - First item',
    children: [
      {
        content: 'Level 2 - Nested under first',
        children: [
          {
            content: 'Level 3 - Deeply nested',
            children: []
          }
        ]
      }
    ]
  },
  {
    content: 'Level 1 - Second item',
    children: [
      {
        content: 'Level 2 - Another nested item',
        children: []
      }
    ]
  }
])

// Alternative: Use insertBlock with sibling parameter
const firstChild = await logseq.Editor.insertBlock(
  parentBlockUUID,
  'First child block',
  { sibling: false }  // Insert as child
)

if (firstChild) {
  // Add nested child
  await logseq.Editor.insertBlock(
    firstChild.uuid,
    'Nested under first child',
    { sibling: false }
  )

  // Add sibling to first child
  await logseq.Editor.insertBlock(
    firstChild.uuid,
    'Sibling to first child',
    { sibling: true }
  )
}
```

**Key Points**:
- No depth limit found (tested 3+ levels)
- `children` array must always be present (use `[]` for leaf nodes)
- Both `insertBatchBlock` and `insertBlock` with `sibling` parameter work
- `insertBatchBlock` is best for creating entire structures at once
- `insertBlock` with `sibling` is best for incremental additions

**Practical Example - Bibliographic Structure**:
```typescript
// Create Zotero-like nested structure
await logseq.Editor.insertBatchBlock(pageUUID, [
  {
    content: '**Abstract:**',
    children: [
      {
        content: htmlToMarkdown(abstract),
        children: []
      }
    ]
  },
  {
    content: '**Authors:**',
    children: authors.map(author => ({
      content: author,
      children: []
    }))
  },
  {
    content: '**Metadata:**',
    children: [
      { content: `Year: ${year}`, children: [] },
      { content: `Journal: ${journal}`, children: [] }
    ]
  }
])
```

### HTML to Markdown Conversion

**CRITICAL**: Logseq displays raw HTML tags literally - they are NOT rendered. Convert HTML to Markdown before insertion.

```typescript
// ✅ RECOMMENDED - Convert HTML to Markdown
function htmlToMarkdown(html: string): string {
  return html
    .replace(/<i>(.*?)<\/i>/g, '*$1*')           // Italic
    .replace(/<em>(.*?)<\/em>/g, '*$1*')         // Emphasis
    .replace(/<b>(.*?)<\/b>/g, '**$1**')         // Bold
    .replace(/<strong>(.*?)<\/strong>/g, '**$1**')  // Strong
    .replace(/<sup>(.*?)<\/sup>/g, '^$1^')       // Superscript
    .replace(/<sub>(.*?)<\/sub>/g, '~$1~')       // Subscript
    .replace(/<a href="(.*?)">(.*?)<\/a>/g, '[$2]($1)')  // Links
    .replace(/<br\s*\/?>/g, '\n')                // Line breaks
    .replace(/<\/?p>/g, '\n')                    // Paragraphs
    .trim()
}

// Use with Zotero abstracts
const markdownAbstract = htmlToMarkdown(zoteroItem.abstractNote)
await logseq.Editor.insertBlock(pageUUID, markdownAbstract)

// ❌ WRONG - Raw HTML shows tags
await logseq.Editor.insertBlock(pageUUID, '<i>italic text</i>')
// Displays: <i>italic text</i> (tags visible)

// ❌ WRONG - Stripped HTML loses formatting
await logseq.Editor.insertBlock(pageUUID, stripHtml('<i>italic text</i>'))
// Displays: italic text (no emphasis)
```

**Markdown Rendering Results**:
- `*text*` → *italic text*
- `**text**` → **bold text**
- `^text^` → <sup>superscript</sup>
- `~text~` → <sub>subscript</sub>
- `[text](url)` → clickable link

### Property Types (DB)

DB graphs support these property types:
- **text** (default): String values - no schema needed
- **number**: Numeric values - `{ type: 'number' }`
- **date**: Date values (YYYY-MM-DD format, links to journal pages) - `{ type: 'date' }`
- **datetime**: Date and time values - `{ type: 'datetime' }`
- **checkbox**: Boolean values - `{ type: 'checkbox' }`
- **url**: URL values (clickable) - `{ type: 'url' }`
- **page** / **node**: References to other pages/blocks - `{ type: 'page' }` or `{ type: 'node' }`
- **json**: Complex objects - `{ type: 'json' }`
- **string**: Explicit string type - `{ type: 'string' }`

### Schema Syntax

```typescript
{
  schema: {
    propertyName: { type: 'propertyType' }
  }
}

// Or shorthand string syntax:
{
  schema: {
    propertyName: 'propertyType'
  }
}
```

### Property Value Validation (DB)

DB graphs validate property values against their schema types:

```typescript
// ✅ CORRECT - value matches schema type
await logseq.Editor.createPage('Page',
  { count: 42 },
  { schema: { count: { type: 'number' } } }
)

// ❌ WRONG - value doesn't match schema type
await logseq.Editor.createPage('Page',
  { count: "42" },  // String instead of number
  { schema: { count: { type: 'number' } } }
)
// Throws error: "should be a number"
```

**Validation Error Messages**:
- `checkbox`: "should be a boolean"
- `number`: Value must be numeric type
- `string`: "should be a string"
- `json`: "should be JSON string"
- `url`: "should be a URL"
- Reference properties validate after entity lookup

**Important**: Validation happens BEFORE database transaction, preventing invalid data from being persisted.

### Multi-value Properties (DB)

Arrays create multi-value properties:

```typescript
// Multi-value text (queryable, not page references)
const page = await logseq.Editor.createPage(
  'Article',
  {
    keywords: ['database', 'knowledge-graph', 'note-taking']
    // No schema needed - defaults to text type
  }
)

// Multi-value page references
const page = await logseq.Editor.createPage(
  'Article',
  {
    relatedPages: ['Page 1', 'Page 2', 'Page 3']
  },
  {
    schema: {
      relatedPages: { type: 'page' }  // Makes them clickable page links
    }
  }
)
```

### Query APIs (DB)

New APIs for querying DB graphs:

```typescript
// Get all tags (classes) in the database
const tags = await logseq.DB.getAllTags()

// Get all properties defined in the database
const properties = await logseq.DB.getAllProperties()

// Get objects tagged with a specific class/tag
// Accepts: UUID string, db/ident, or tag title (case-insensitive)
const objects = await logseq.DB.getTagObjects('tag-uuid')
const objects = await logseq.DB.getTagObjects('Tag Name')  // Case-insensitive
```

**Important**: `getTagObjects` validates that the input resolves to an actual tag/class and throws an error if it doesn't.

## Common Patterns

### Error Handling
```typescript
try {
  const page = await logseq.Editor.createPage(pageName)
  if (!page) {
    throw new Error('Page creation returned null')
  }
  // ... rest of logic
} catch (error) {
  console.error('Plugin error:', error)
  logseq.UI.showMsg(
    `Error: ${error instanceof Error ? error.message : String(error)}`,
    'error'
  )
}
```

### User Notifications
```typescript
// Info message
logseq.UI.showMsg('Operation in progress...', 'info')

// Success message
logseq.UI.showMsg('Operation completed!', 'success', { timeout: 5000 })

// Error message
logseq.UI.showMsg('Operation failed!', 'error', { timeout: 5000 })

// Warning message
logseq.UI.showMsg('Warning: check your settings', 'warning')
```

### Navigation
```typescript
// Navigate to a page
logseq.App.pushState('page', { name: pageName })

// Navigate to a block
logseq.App.pushState('block', { uuid: blockUUID })
```

## Build Configuration

### vite.config.ts
```typescript
import { defineConfig } from 'vite'

export default defineConfig({
  build: {
    target: 'esnext',
    minify: 'esbuild',
    rollupOptions: {
      external: ['@logseq/libs']
    }
  }
})
```

### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "lib": ["ESNext", "DOM"],
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"]
}
```

## Testing & Development

### Build Commands
```bash
# Install dependencies
pnpm install

# Build plugin
pnpm build

# Development mode (if configured)
pnpm dev
```

### Loading in Logseq
1. Open Logseq
2. Settings → Plugins → Load unpacked plugin
3. Select plugin directory
4. Plugin will appear in toolbar and slash commands

### Debugging
- Use browser DevTools (Cmd+Opt+I on Mac)
- Check console for errors and logs
- Use `console.log()` for debugging
- Check Logseq's plugin console for errors

### Exposing Functions to Browser Console (POC 6)

**Problem**: Logseq plugins run in an iframe context, so setting functions on `window` doesn't make them accessible from the browser console.

**Solution**: Expose functions to parent and top windows:

```typescript
async function main() {
  // Your plugin code...

  // Expose functions for console access
  // @ts-ignore
  globalThis.myFunction = myFunction

  // For iframe context (Logseq plugins)
  // @ts-ignore
  if (typeof parent !== 'undefined' && parent.window) {
    // @ts-ignore
    parent.window.myFunction = myFunction
  }

  // @ts-ignore
  if (typeof top !== 'undefined' && top.window && top !== window) {
    // @ts-ignore
    top.window.myFunction = myFunction
  }
}
```

**Usage**: Now you can call from browser console:
```javascript
await window.myFunction()
```

**Note**: This is useful for POC testing. Production plugins should use proper UI instead of console commands.

### Checking for Duplicates (POC 6)

**Best Practice**: Query by unique identifier property, not by page title.

```typescript
// ✅ BEST: Query by unique property
async function checkIfExists(zoteroKey: string): Promise<boolean> {
  try {
    // Query database for pages with this property
    const query = `(property zoteroKey "${zoteroKey}")`
    const results = await logseq.DB.q(query)
    return !!(results && results.length > 0)
  } catch (error) {
    console.warn('Query failed:', error)
    return false
  }
}

// ❌ AVOID: Title-based checking (false positives)
async function checkIfExistsByTitle(title: string): Promise<boolean> {
  const page = await logseq.Editor.getPage(title)
  return !!page
}
```

**Why query by property?**
- Most reliable: unique identifiers prevent false positives
- Works even if page title is modified
- Handles duplicate titles correctly

**Common use cases**:
- Zotero import: check by `zoteroKey`
- External IDs: check by `externalId`, `doi`, `isbn`, etc.
- Any unique identifier property

## Resources

- Official Plugin API: https://logseq.github.io/plugins/
- Plugin Samples: https://github.com/logseq/logseq-plugin-samples
- @logseq/libs documentation: https://logseq.github.io/plugins/
- Community plugins: https://github.com/logseq/marketplace

## Known Issues & Gotchas

1. **MD vs DB**: Plugins designed for markdown graphs may not work correctly with DB graphs
2. **Schema Parameter Broken**: The `schema` parameter in `createPage` causes `DataCloneError: function could not be cloned` - DO NOT USE in @logseq/libs v0.0.17
3. **Type Inference**: Types are auto-inferred from JavaScript values - use actual numbers/booleans, not strings
4. **Silent Property Dropping**: When a property fails validation, ALL subsequent properties in the object are silently dropped - always check console for validation errors
5. **Reserved Property Names**: Properties named `created`, `modified`, etc. trigger special validation rules (e.g., "should be a journal date")
6. **Date Properties**:
   - Reserved names like `created`/`modified` expect "journal date" format (page references)
   - Both `"YYYY-MM-DD"` strings and Unix timestamps fail validation
   - **Workaround**: Use different names (`dateCreated`, `publicationDate`) or store as text
7. **Property Namespacing**: Plugin properties are auto-namespaced as `:plugin.property.{plugin-id}/{property-name}`
8. **UUID Auto-generation**: When blocks lack UUIDs, the system auto-generates them during insertion
9. **Case-Insensitive Tags**: Tag lookup by title is case-insensitive via `ldb/get-case-page`
10. **Property Schemas on Tags - UI-ONLY** (POC 4):
    - Property schemas CANNOT be defined programmatically on tag/class pages
    - Requires `:logseq.property.class/properties` with db/id references
    - Plugin API restricts: "Plugins can only upsert its own properties"
    - No API exists to create property definition entities
    - **Workaround**: Use type inference from JavaScript values (recommended approach)

## References

This skill is based on official Logseq source code and tests:

**Primary Sources**:
1. Plugin API test file: `https://github.com/logseq/logseq/blob/master/clj-e2e/test/logseq/e2e/plugins_basic_test.clj`
   - Contains working examples of createPage, insertBlock, insertBatchBlock with schema
2. DB Graph API commit: `https://github.com/logseq/logseq/commit/4800ed2`
   - Added getAllTags, getAllProperties, getTagObjects APIs
3. ClojureScript SDK commit: `https://github.com/logseq/logseq/commit/d8809f0`
   - Added ClojureScript wrappers for plugin SDK
4. API refactoring commit: `https://github.com/logseq/logseq/commit/15de4a8`
   - Separated API into namespace modules, clarified property/schema structure
5. Tag objects enhancement: `https://github.com/logseq/logseq/commit/3962f1f`
   - Enhanced getTagObjects to accept UUID, ident, or title
6. Property validation commit: `https://github.com/logseq/logseq/commit/dbd15f7`
   - Added pre-validation for property values with readable error messages
7. DB version documentation: `https://github.com/logseq/docs/blob/master/db-version-changes.md`
   - Official documentation about DB graph differences

**Key Insight**: The test file `plugins_basic_test.clj` shows the INTENDED API, but the `schema` parameter doesn't work in practice with @logseq/libs v0.0.17 due to serialization issues.

## Verified Working Examples (Tested in POC)

### Complete Working Example

```typescript
// VERIFIED WORKING - POC 1 tested successfully
const page = await logseq.Editor.createPage(
  'My Article',
  {
    // Text properties (default type)
    title: 'Article Title',
    description: 'Description text',
    author1: 'Jane Doe',
    author2: 'John Smith',

    // Number property (actual number, not string)
    year: 2025,

    // Boolean property (actual boolean)
    verified: true,

    // URL property (auto-detected from string)
    link: 'https://example.com',

    // Multi-value properties (arrays become sets)
    tags: ['research', 'science', 'database'],
    categories: ['academic', 'published'],

    // Empty string (auto-hides in UI)
    notes: ''
  },
  {
    redirect: false,
    createFirstBlock: false
  }
)

// Result: All properties created with correct types
// - year: 2025 (number)
// - verified: true (boolean)
// - tags: #{"research" "science" "database"} (set)
// - Properties namespaced: :plugin.property.{plugin-id}/propertyName
```

### What NOT to Do

```typescript
// ❌ WRONG - Using schema parameter
const page = await logseq.Editor.createPage(
  'Page',
  { year: 2025 },
  { schema: { year: { type: 'number' } } }  // Causes DataCloneError
)

// ❌ WRONG - Using reserved property names for dates
const page = await logseq.Editor.createPage(
  'Page',
  {
    created: '2025-11-01',  // Fails validation, drops ALL subsequent properties
    title: 'Will be lost'   // Never created due to validation failure above
  }
)

// ❌ WRONG - Using strings for numbers/booleans
const page = await logseq.Editor.createPage(
  'Page',
  {
    year: '2025',      // String, not number
    verified: 'true'   // String, not boolean
  }
)
```

## Version Information

- **@logseq/libs**: v0.0.17 (tested with POC)
- **DB Graphs**: Available in Logseq 0.10.0+
- **Status**: DB graphs are in beta - API may evolve
- **Schema support**: Documented but broken in current version

## Migration from MD to DB Plugins

If migrating an existing markdown-based plugin to DB graphs:

1. **Update createPage calls**: Add schema parameter for typed properties
2. **Update insertBlock calls**: Add schema parameter for block properties
3. **Update insertBatchBlock calls**: Add schema parameter to options object
4. **Test thoroughly**: MD-style syntax may appear to work but creates incorrect property structure
5. **Check property types**: Ensure numeric, boolean, URL, and page reference properties have correct schema declarations

**Common Migration Issues**:
- Properties appearing in page title instead of property section → Testing on MD graph instead of DB graph
- Properties showing as text when they should be numbers/booleans → Using string values instead of actual JavaScript types
- All properties missing after first one → Property validation failure causing silent dropping
- DataCloneError on createPage → Using schema parameter (broken in v0.0.17)
- Empty properties not auto-hiding → May indicate MD-style properties instead of DB-style

## Summary

**For DB Graph Plugin Development (as of @logseq/libs v0.0.17)**:

✅ **Works**:
- Creating pages with properties (without schema)
- Auto-type inference from JavaScript values
- Number, boolean, URL, text, multi-value array properties
- Property auto-namespacing
- Empty property auto-hiding
- Creating tag/class pages with `{ class: true }`
- Tagging pages via `#tag` in page title
- Nested block structures with `insertBatchBlock`
- HTML to Markdown conversion
- Database queries with `logseq.DB.q()` for duplicate detection
- Exposing functions to browser console via parent/top window

❌ **Broken/Issues**:
- Schema parameter causes DataCloneError
- Date properties with reserved names fail validation
- Property validation failures silently drop all subsequent properties
- No way to create page references without schema
- **Property schemas on tags - UI-only, no plugin API**

**Best Practices**:
1. Use actual JavaScript types (numbers, booleans) not strings
2. Avoid reserved property names (`created`, `modified`)
3. Test on actual DB graphs, not MD graphs
4. Check browser console for validation errors
5. Put most important properties first (validation failures drop subsequent ones)
6. **Use type inference instead of attempting schema definition**
7. Include tags in page title using `#tag` syntax
8. Convert HTML to Markdown for proper formatting
9. **Check for duplicates using `logseq.DB.q()` with unique property, not page title**
10. **Expose test functions to parent/top window for iframe context**
