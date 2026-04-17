# AST Query Tools Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 4 new AST query tools (find_implementations, get_class_code, search_symbols, get_file_symbols) with both MCP and CLI support.

**Architecture:** Add tool definitions to `tools.ts`, implement backend methods in `local-backend.ts`, register in `server.ts`, and add CLI commands in `cli/tool.ts` and `cli/index.ts`. Each tool uses Cypher queries against the LadybugDB graph.

**Tech Stack:** TypeScript, LadybugDB (Cypher), Vitest for testing

**Spec:** `docs/superpowers/specs/2026-04-17-ast-query-tools-design.md`

---

## File Structure

| File | Purpose |
|------|---------|
| `gitnexus/src/mcp/tools.ts` | Tool definitions (input schema, description) |
| `gitnexus/src/mcp/local/local-backend.ts` | Backend implementation (Cypher queries) |
| `gitnexus/src/mcp/server.ts` | Tool registration |
| `gitnexus/src/cli/tool.ts` | CLI command functions |
| `gitnexus/src/cli/index.ts` | CLI command registration |
| `gitnexus/test/unit/mcp/ast-query-tools.test.ts` | Unit tests |

---

## Task 1: Add Tool Definitions to tools.ts

**Files:**
- Modify: `gitnexus/src/mcp/tools.ts`

- [ ] **Step 1: Add find_implementations tool definition**

Add after `group_status` tool definition (around line 455):

```typescript
  {
    name: 'find_implementations',
    description: `Find all classes that implement a given interface or extend a base class.

WHEN TO USE: When you need to find concrete implementations of an interface or subclasses of a base class.
AFTER THIS: Use get_class_code to retrieve full implementation details.

Returns: List of implementing classes with their file locations, and optionally the full code.`,
    inputSchema: {
      type: 'object',
      properties: {
        interface_name: { type: 'string', description: 'Interface or base class name to find implementations for' },
        fuzzy_match: { type: 'boolean', description: 'Enable fuzzy matching (CONTAINS) for interface name (default: true)', default: true },
        include_content: { type: 'boolean', description: 'Include full source code for each implementation (default: false)', default: false },
        repo: { type: 'string', description: 'Repository name or path. Omit if only one repo is indexed.' },
      },
      required: ['interface_name'],
    },
  },
```

- [ ] **Step 2: Add get_class_code tool definition**

Add after `find_implementations`:

```typescript
  {
    name: 'get_class_code',
    description: `Get complete class code including methods, fields, and relationships.

WHEN TO USE: When you need the full source code of a class, including its methods and fields.
AFTER THIS: Use impact() to analyze what depends on this class.

Returns: Class metadata, full source code, method list, field list, implements/extends relationships.`,
    inputSchema: {
      type: 'object',
      properties: {
        class_name: { type: 'string', description: 'Class name (supports fuzzy matching)' },
        file_path: { type: 'string', description: 'File path for exact match (alternative to class_name)' },
        include_methods: { type: 'boolean', description: 'Include method list (default: true)', default: true },
        include_fields: { type: 'boolean', description: 'Include field list (default: true)', default: true },
        include_content: { type: 'boolean', description: 'Include full source code (default: true)', default: true },
        repo: { type: 'string', description: 'Repository name or path. Omit if only one repo is indexed.' },
      },
      required: [],
    },
  },
```

- [ ] **Step 3: Add search_symbols tool definition**

Add after `get_class_code`:

```typescript
  {
    name: 'search_symbols',
    description: `Fuzzy search symbols by name or keyword in code content.

WHEN TO USE: When you need to find symbols (classes, methods, functions) by name or search for code patterns.
AFTER THIS: Use context() on a specific symbol for deeper analysis.

Returns: List of matching symbols with their types, file locations, and optionally source code.`,
    inputSchema: {
      type: 'object',
      properties: {
        query: { type: 'string', description: 'Search keyword or code snippet' },
        symbol_type: {
          type: 'string',
          description: 'Filter by symbol type',
          enum: ['Class', 'Interface', 'Method', 'Function', 'all'],
          default: 'all',
        },
        file_path: { type: 'string', description: 'Filter by file path (partial match)' },
        fuzzy_match: { type: 'boolean', description: 'Enable fuzzy matching (default: true)', default: true },
        search_content: { type: 'boolean', description: 'Search in code content, not just symbol names (default: false)', default: false },
        include_content: { type: 'boolean', description: 'Include source code in results (default: false)', default: false },
        limit: { type: 'number', description: 'Maximum number of results (default: 20)', default: 20 },
        repo: { type: 'string', description: 'Repository name or path. Omit if only one repo is indexed.' },
      },
      required: ['query'],
    },
  },
```

- [ ] **Step 4: Add get_file_symbols tool definition**

Add after `search_symbols`:

```typescript
  {
    name: 'get_file_symbols',
    description: `Get all symbols (classes, methods, functions) in a file by file path.

WHEN TO USE: When you need to see all code structures in a specific file.
AFTER THIS: Use get_class_code or context() for detailed analysis of specific symbols.

Returns: File metadata and list of all symbols with their types, line numbers, and optionally source code.`,
    inputSchema: {
      type: 'object',
      properties: {
        file_path: { type: 'string', description: 'File path (supports partial match)' },
        include_content: { type: 'boolean', description: 'Include source code for each symbol (default: false)', default: false },
        symbol_type: {
          type: 'string',
          description: 'Filter by symbol type',
          enum: ['Class', 'Interface', 'Method', 'Function', 'all'],
          default: 'all',
        },
        repo: { type: 'string', description: 'Repository name or path. Omit if only one repo is indexed.' },
      },
      required: ['file_path'],
    },
  },
```

- [ ] **Step 5: Run typecheck to verify tool definitions**

Run: `cd gitnexus && npx tsc --noEmit`
Expected: No errors

---

## Task 2: Implement Backend Methods in local-backend.ts

**Files:**
- Modify: `gitnexus/src/mcp/local/local-backend.ts`

- [ ] **Step 1: Add findImplementations method**

Add after the `groupStatus` method (around line 2950):

```typescript
  private async findImplementations(
    repo: RepoHandle,
    params: {
      interface_name: string;
      fuzzy_match?: boolean;
      include_content?: boolean;
    },
  ): Promise<any> {
    await this.ensureInitialized(repo.id);

    const { interface_name, fuzzy_match = true, include_content = false } = params;

    if (!interface_name?.trim()) {
      return { error: 'interface_name parameter is required.' };
    }

    const nameFilter = fuzzy_match
      ? 'WHERE iface.name CONTAINS $name'
      : 'WHERE iface.name = $name';

    // Find implementations via IMPLEMENTS relationship
    const implRows = await executeParameterized(
      repo.id,
      `
      MATCH (impl)-[r:CodeRelation {type: 'IMPLEMENTS'}]->(iface)
      ${nameFilter}
      RETURN impl.id AS id, impl.name AS name, labels(impl)[0] AS type, 
             impl.filePath AS filePath, impl.startLine AS startLine, 
             impl.endLine AS endLine, iface.name AS interfaceName
             ${include_content ? ', impl.content AS content' : ''}
      `,
      { name: interface_name },
    ).catch(() => []);

    // Also check EXTENDS relationship for base classes
    const extendsRows = await executeParameterized(
      repo.id,
      `
      MATCH (child)-[r:CodeRelation {type: 'EXTENDS'}]->(parent)
      ${nameFilter.replace(/iface/g, 'parent')}
      RETURN child.id AS id, child.name AS name, labels(child)[0] AS type,
             child.filePath AS filePath, child.startLine AS startLine,
             child.endLine AS endLine, parent.name AS interfaceName
             ${include_content ? ', child.content AS content' : ''}
      `,
      { name: interface_name },
    ).catch(() => []);

    const allImpls = [...implRows, ...extendsRows];

    if (allImpls.length === 0) {
      return {
        interface: interface_name,
        implementations: [],
        total: 0,
        message: `No implementations found for "${interface_name}".`,
      };
    }

    return {
      interface: interface_name,
      implementations: allImpls.map((row) => ({
        name: row.name || row[1],
        type: row.type || row[2],
        filePath: row.filePath || row[3],
        startLine: row.startLine || row[4],
        endLine: row.endLine || row[5],
        interfaceName: row.interfaceName || row[6],
        ...(include_content && row.content ? { content: row.content || row[7] } : {}),
      })),
      total: allImpls.length,
    };
  }
```

- [ ] **Step 2: Add getClassCode method**

Add after `findImplementations`:

```typescript
  private async getClassCode(
    repo: RepoHandle,
    params: {
      class_name?: string;
      file_path?: string;
      include_methods?: boolean;
      include_fields?: boolean;
      include_content?: boolean;
    },
  ): Promise<any> {
    await this.ensureInitialized(repo.id);

    const { class_name, file_path, include_methods = true, include_fields = true, include_content = true } = params;

    if (!class_name && !file_path) {
      return { error: 'Either class_name or file_path parameter is required.' };
    }

    // Build filter clause
    let filterClause: string;
    let queryParams: Record<string, any>;

    if (file_path) {
      filterClause = 'WHERE c.filePath CONTAINS $filePath';
      queryParams = { filePath: file_path };
    } else {
      filterClause = 'WHERE c.name = $name OR c.name CONTAINS $name';
      queryParams = { name: class_name };
    }

    // Find the class
    const classRows = await executeParameterized(
      repo.id,
      `
      MATCH (c:Class)
      ${filterClause}
      RETURN c.id AS id, c.name AS name, c.filePath AS filePath, 
             c.startLine AS startLine, c.endLine AS endLine,
             c.isExported AS isExported
             ${include_content ? ', c.content AS content' : ''}
      LIMIT 1
      `,
      queryParams,
    ).catch(() => []);

    if (classRows.length === 0) {
      return {
        error: `Class not found: ${class_name || file_path}`,
        suggestion: 'Try using search_symbols to find the correct class name.',
      };
    }

    const classInfo = classRows[0];
    const classId = classInfo.id || classInfo[0];

    const result: any = {
      class: {
        name: classInfo.name || classInfo[1],
        type: 'Class',
        filePath: classInfo.filePath || classInfo[2],
        startLine: classInfo.startLine || classInfo[3],
        endLine: classInfo.endLine || classInfo[4],
        isExported: classInfo.isExported || classInfo[5],
        ...(include_content && classInfo.content ? { content: classInfo.content || classInfo[6] } : {}),
      },
      methods: [],
      fields: [],
      implements: [],
      extends: null,
    };

    // Get methods
    if (include_methods) {
      const methodRows = await executeParameterized(
        repo.id,
        `
        MATCH (c:Class {id: $classId})-[r:CodeRelation {type: 'HAS_METHOD'}]->(m:Method)
        RETURN m.name AS name, m.startLine AS startLine, m.endLine AS endLine,
               m.parameterCount AS parameterCount, m.returnType AS returnType
               ${include_content ? ', m.content AS content' : ''}
        ORDER BY m.startLine
        `,
        { classId },
      ).catch(() => []);

      result.methods = methodRows.map((row) => ({
        name: row.name || row[0],
        startLine: row.startLine || row[1],
        endLine: row.endLine || row[2],
        parameterCount: row.parameterCount || row[3],
        returnType: row.returnType || row[4],
        ...(include_content && row.content ? { content: row.content || row[5] } : {}),
      }));
    }

    // Get fields/properties
    if (include_fields) {
      const fieldRows = await executeParameterized(
        repo.id,
        `
        MATCH (c:Class {id: $classId})-[r:CodeRelation {type: 'HAS_PROPERTY'}]->(p:Property)
        RETURN p.name AS name, p.declaredType AS declaredType, p.startLine AS startLine
        ORDER BY p.startLine
        `,
        { classId },
      ).catch(() => []);

      result.fields = fieldRows.map((row) => ({
        name: row.name || row[0],
        type: row.declaredType || row[1],
        startLine: row.startLine || row[2],
      }));
    }

    // Get implements
    const implRows = await executeParameterized(
      repo.id,
      `
      MATCH (c:Class {id: $classId})-[r:CodeRelation {type: 'IMPLEMENTS'}]->(i:Interface)
      RETURN i.name AS name
      `,
      { classId },
    ).catch(() => []);

    result.implements = implRows.map((row) => row.name || row[0]);

    // Get extends
    const extendsRows = await executeParameterized(
      repo.id,
      `
      MATCH (c:Class {id: $classId})-[r:CodeRelation {type: 'EXTENDS'}]->(parent)
      RETURN parent.name AS name
      LIMIT 1
      `,
      { classId },
    ).catch(() => []);

    if (extendsRows.length > 0) {
      result.extends = extendsRows[0].name || extendsRows[0][0];
    }

    return result;
  }
```

- [ ] **Step 3: Add searchSymbols method**

Add after `getClassCode`:

```typescript
  private async searchSymbols(
    repo: RepoHandle,
    params: {
      query: string;
      symbol_type?: string;
      file_path?: string;
      fuzzy_match?: boolean;
      search_content?: boolean;
      include_content?: boolean;
      limit?: number;
    },
  ): Promise<any> {
    await this.ensureInitialized(repo.id);

    const {
      query: searchQuery,
      symbol_type = 'all',
      file_path,
      fuzzy_match = true,
      search_content = false,
      include_content = false,
      limit = 20,
    } = params;

    if (!searchQuery?.trim()) {
      return { error: 'query parameter is required.' };
    }

    // Build type filter
    const typeFilter = symbol_type === 'all' ? '' : `AND labels(n)[0] = $symbolType`;

    // Build file filter
    const fileFilter = file_path ? 'AND n.filePath CONTAINS $filePath' : '';

    // Build name filter
    const nameFilter = fuzzy_match ? 'n.name CONTAINS $query' : 'n.name = $query';

    // Build content filter
    const contentFilter = search_content ? `OR n.content CONTAINS $query` : '';

    const queryParams: Record<string, any> = { query: searchQuery };
    if (symbol_type !== 'all') queryParams.symbolType = symbol_type;
    if (file_path) queryParams.filePath = file_path;

    const rows = await executeParameterized(
      repo.id,
      `
      MATCH (n)
      WHERE (${nameFilter} ${contentFilter})
        ${typeFilter}
        ${fileFilter}
        AND labels(n)[0] IN ['Class', 'Interface', 'Method', 'Function', 'Constructor', 'Struct', 'Enum', 'Trait']
      RETURN n.id AS id, n.name AS name, labels(n)[0] AS type,
             n.filePath AS filePath, n.startLine AS startLine, n.endLine AS endLine
             ${include_content ? ', n.content AS content' : ''}
      LIMIT $limit
      `,
      { ...queryParams, limit },
    ).catch(() => []);

    return {
      query: searchQuery,
      results: rows.map((row) => ({
        name: row.name || row[1],
        type: row.type || row[2],
        filePath: row.filePath || row[3],
        startLine: row.startLine || row[4],
        endLine: row.endLine || row[5],
        matchType: 'fuzzy',
        ...(include_content && row.content ? { content: row.content || row[6] } : {}),
      })),
      total: rows.length,
    };
  }
```

- [ ] **Step 4: Add getFileSymbols method**

Add after `searchSymbols`:

```typescript
  private async getFileSymbols(
    repo: RepoHandle,
    params: {
      file_path: string;
      include_content?: boolean;
      symbol_type?: string;
    },
  ): Promise<any> {
    await this.ensureInitialized(repo.id);

    const { file_path, include_content = false, symbol_type = 'all' } = params;

    if (!file_path?.trim()) {
      return { error: 'file_path parameter is required.' };
    }

    // Find the file
    const fileRows = await executeParameterized(
      repo.id,
      `
      MATCH (f:File)
      WHERE f.filePath CONTAINS $path OR f.name = $path
      RETURN f.id AS id, f.name AS name, f.filePath AS filePath
      LIMIT 1
      `,
      { path: file_path },
    ).catch(() => []);

    if (fileRows.length === 0) {
      return {
        error: `File not found: ${file_path}`,
        suggestion: 'Try using a partial file name or path.',
      };
    }

    const fileInfo = fileRows[0];
    const fileId = fileInfo.id || fileInfo[0];

    // Build type filter
    const typeFilter = symbol_type === 'all' ? '' : `AND labels(n)[0] = $symbolType`;
    const queryParams: Record<string, any> = { fileId };
    if (symbol_type !== 'all') queryParams.symbolType = symbol_type;

    // Get all symbols defined in this file
    const symbolRows = await executeParameterized(
      repo.id,
      `
      MATCH (f:File {id: $fileId})-[r:CodeRelation {type: 'DEFINES'}]->(n)
      WHERE labels(n)[0] IN ['Class', 'Interface', 'Method', 'Function', 'Constructor', 'Struct', 'Enum', 'Trait', 'Const', 'Property']
        ${typeFilter}
      RETURN n.id AS id, n.name AS name, labels(n)[0] AS type,
             n.filePath AS filePath, n.startLine AS startLine, n.endLine AS endLine,
             n.isExported AS isExported
             ${include_content ? ', n.content AS content' : ''}
      ORDER BY n.startLine
      `,
      queryParams,
    ).catch(() => []);

    return {
      file: {
        name: fileInfo.name || fileInfo[1],
        filePath: fileInfo.filePath || fileInfo[2],
      },
      symbols: symbolRows.map((row) => ({
        name: row.name || row[1],
        type: row.type || row[2],
        filePath: row.filePath || row[3],
        startLine: row.startLine || row[4],
        endLine: row.endLine || row[5],
        isExported: row.isExported || row[6],
        ...(include_content && row.content ? { content: row.content || row[7] } : {}),
      })),
      total: symbolRows.length,
    };
  }
```

- [ ] **Step 5: Add case handlers in callTool method**

Find the `callTool` method in `local-backend.ts` and add cases for the new tools. Look for the existing switch statement around line 300:

```typescript
    switch (tool) {
      // ... existing cases ...
      
      case 'find_implementations':
        return this.findImplementations(repo, args as any);
      
      case 'get_class_code':
        return this.getClassCode(repo, args as any);
      
      case 'search_symbols':
        return this.searchSymbols(repo, args as any);
      
      case 'get_file_symbols':
        return this.getFileSymbols(repo, args as any);
      
      // ... rest of switch ...
    }
```

- [ ] **Step 6: Run typecheck to verify backend methods**

Run: `cd gitnexus && npx tsc --noEmit`
Expected: No errors

---

## Task 3: Register Tools in server.ts

**Files:**
- Modify: `gitnexus/src/mcp/server.ts`

- [ ] **Step 1: Verify tools are automatically registered**

Check if `server.ts` iterates over `GITNEXUS_TOOLS` array. If so, tools are auto-registered and no changes needed.

Run: `grep -n "GITNEXUS_TOOLS" gitnexus/src/mcp/server.ts`

If output shows iteration over the array, skip this task. Otherwise, add manual registration.

- [ ] **Step 2: Run typecheck**

Run: `cd gitnexus && npx tsc --noEmit`
Expected: No errors

---

## Task 4: Add CLI Commands to tool.ts

**Files:**
- Modify: `gitnexus/src/cli/tool.ts`

- [ ] **Step 1: Add findImplementationsCommand**

Add after `cypherCommand` (around line 165):

```typescript
export async function findImplementationsCommand(
  interfaceName: string,
  options?: {
    repo?: string;
    fuzzy?: boolean;
    content?: boolean;
  },
): Promise<void> {
  if (!interfaceName?.trim()) {
    console.error('Usage: gitnexus find-impl <interface_name>');
    process.exit(1);
  }

  const backend = await getBackend();
  const result = await backend.callTool('find_implementations', {
    interface_name: interfaceName,
    fuzzy_match: options?.fuzzy ?? true,
    include_content: options?.content ?? false,
    repo: options?.repo,
  });
  output(result);
}
```

- [ ] **Step 2: Add getClassCodeCommand**

Add after `findImplementationsCommand`:

```typescript
export async function getClassCodeCommand(
  className: string,
  options?: {
    repo?: string;
    file?: string;
    methods?: boolean;
    fields?: boolean;
    content?: boolean;
  },
): Promise<void> {
  if (!className?.trim() && !options?.file) {
    console.error('Usage: gitnexus class-code <class_name> [--file <path>]');
    process.exit(1);
  }

  const backend = await getBackend();
  const result = await backend.callTool('get_class_code', {
    class_name: className || undefined,
    file_path: options?.file,
    include_methods: options?.methods ?? true,
    include_fields: options?.fields ?? true,
    include_content: options?.content ?? true,
    repo: options?.repo,
  });
  output(result);
}
```

- [ ] **Step 3: Add searchSymbolsCommand**

Add after `getClassCodeCommand`:

```typescript
export async function searchSymbolsCommand(
  query: string,
  options?: {
    repo?: string;
    type?: string;
    file?: string;
    fuzzy?: boolean;
    contentSearch?: boolean;
    content?: boolean;
    limit?: string;
  },
): Promise<void> {
  if (!query?.trim()) {
    console.error('Usage: gitnexus search <query>');
    process.exit(1);
  }

  const backend = await getBackend();
  const result = await backend.callTool('search_symbols', {
    query,
    symbol_type: options?.type || 'all',
    file_path: options?.file,
    fuzzy_match: options?.fuzzy ?? true,
    search_content: options?.contentSearch ?? false,
    include_content: options?.content ?? false,
    limit: options?.limit ? parseInt(options.limit) : 20,
    repo: options?.repo,
  });
  output(result);
}
```

- [ ] **Step 4: Add getFileSymbolsCommand**

Add after `searchSymbolsCommand`:

```typescript
export async function getFileSymbolsCommand(
  filePath: string,
  options?: {
    repo?: string;
    type?: string;
    content?: boolean;
  },
): Promise<void> {
  if (!filePath?.trim()) {
    console.error('Usage: gitnexus file-symbols <file_path>');
    process.exit(1);
  }

  const backend = await getBackend();
  const result = await backend.callTool('get_file_symbols', {
    file_path: filePath,
    include_content: options?.content ?? false,
    symbol_type: options?.type || 'all',
    repo: options?.repo,
  });
  output(result);
}
```

- [ ] **Step 5: Run typecheck**

Run: `cd gitnexus && npx tsc --noEmit`
Expected: No errors

---

## Task 5: Register CLI Commands in index.ts

**Files:**
- Modify: `gitnexus/src/cli/index.ts`

- [ ] **Step 1: Import new command functions**

Find the import section for `tool.ts` and add the new commands:

```typescript
import {
  queryCommand,
  contextCommand,
  impactCommand,
  cypherCommand,
  findImplementationsCommand,
  getClassCodeCommand,
  searchSymbolsCommand,
  getFileSymbolsCommand,
} from './tool.js';
```

- [ ] **Step 2: Register find-impl command**

Find the command registration section and add:

```typescript
program
  .command('find-impl <interface_name>')
  .description('Find classes implementing an interface or extending a base class')
  .option('--no-fuzzy', 'Disable fuzzy matching (exact match only)')
  .option('--content', 'Include source code in results')
  .option('--repo <repo>', 'Repository name or path')
  .action((interfaceName, options) => findImplementationsCommand(interfaceName, options));
```

- [ ] **Step 3: Register class-code command**

Add after `find-impl`:

```typescript
program
  .command('class-code [class_name]')
  .description('Get complete class code with methods and fields')
  .option('--file <path>', 'File path for exact match')
  .option('--no-methods', 'Exclude method list')
  .option('--no-fields', 'Exclude field list')
  .option('--no-content', 'Exclude source code (metadata only)')
  .option('--repo <repo>', 'Repository name or path')
  .action((className, options) => getClassCodeCommand(className, options));
```

- [ ] **Step 4: Register search command**

Add after `class-code`:

```typescript
program
  .command('search <query>')
  .description('Search symbols by name or keyword')
  .option('--type <type>', 'Filter by symbol type (Class, Interface, Method, Function, all)', 'all')
  .option('--file <path>', 'Filter by file path')
  .option('--no-fuzzy', 'Disable fuzzy matching')
  .option('--content-search', 'Search in code content')
  .option('--content', 'Include source code in results')
  .option('--limit <n>', 'Maximum results', '20')
  .option('--repo <repo>', 'Repository name or path')
  .action((query, options) => searchSymbolsCommand(query, options));
```

- [ ] **Step 5: Register file-symbols command**

Add after `search`:

```typescript
program
  .command('file-symbols <file_path>')
  .description('Get all symbols in a file')
  .option('--type <type>', 'Filter by symbol type (Class, Interface, Method, Function, all)', 'all')
  .option('--content', 'Include source code for each symbol')
  .option('--repo <repo>', 'Repository name or path')
  .action((filePath, options) => getFileSymbolsCommand(filePath, options));
```

- [ ] **Step 6: Run typecheck**

Run: `cd gitnexus && npx tsc --noEmit`
Expected: No errors

---

## Task 6: Run Tests and Verify

- [ ] **Step 1: Run unit tests**

Run: `cd gitnexus && npm test`
Expected: All existing tests pass

- [ ] **Step 2: Build the project**

Run: `cd gitnexus && npm run build`
Expected: Build succeeds

- [ ] **Step 3: Test CLI commands manually**

```bash
# Test find-impl
gitnexus find-impl --interface "Service"

# Test class-code
gitnexus class-code --name "LocalBackend"

# Test search
gitnexus search --query "query" --type Method

# Test file-symbols
gitnexus file-symbols --path "local-backend.ts"
```

Expected: Each command returns valid JSON output

- [ ] **Step 4: Commit changes**

```bash
cd gitnexus && git add src/mcp/tools.ts src/mcp/local/local-backend.ts src/mcp/server.ts src/cli/tool.ts src/cli/index.ts
git commit -m "feat: add 4 AST query tools (find_implementations, get_class_code, search_symbols, get_file_symbols)

Add MCP tools and CLI commands for:
- find-impl: Find classes implementing an interface
- class-code: Get complete class code with methods/fields
- search: Fuzzy search symbols by name or keyword
- file-symbols: Get all symbols in a file

Both MCP and CLI interfaces are supported.

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Out of Scope

- Unit tests for new tools (can be added later)
- Web UI support
- Streaming responses
- Performance optimization/caching

---

## Validation Commands

After implementation:
1. `cd gitnexus && npx tsc --noEmit` - Type check
2. `cd gitnexus && npm test` - Run tests
3. `cd gitnexus && npm run build` - Build
4. Manual CLI testing with indexed repo