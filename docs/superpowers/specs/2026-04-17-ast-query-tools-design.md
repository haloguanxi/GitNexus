# AST Query Tools Design

**Date:** 2026-04-17
**Author:** User request
**Scope:** Add 4 new tools for AST-based code queries (CLI + MCP)

---

## Background

User needs to use GitNexus's AST parsing capabilities to build a local knowledge graph with fast AST queries. Current GitNexus tools are designed for AI agents (semantic search, impact analysis) but lack direct symbol lookup and code retrieval capabilities.

---

## Requirements Summary

| # | Requirement | Current Support | Gap |
|---|-------------|-----------------|-----|
| 1 | Find implementations by interface/class name (fuzzy search) | Partial - requires Cypher | No dedicated tool, no fuzzy search |
| 2 | Get full class code by class name | Partial - `context()` returns partial info | No complete code retrieval |
| 3 | Get downstream dependencies by method name | Full - `impact()` | N/A |
| 4 | Get method content by method name | Full - `context()` | N/A |
| 5 | Search code keywords to find containing class | Partial - `query()` is semantic | No exact keyword search |
| 6 | Get all symbols in a file by file path | None | New capability |

---

## Design

### New Tools Overview

| Tool | CLI Command | Purpose |
|------|-------------|---------|
| `find_implementations` | `gitnexus find-impl` | Find classes implementing an interface |
| `get_class_code` | `gitnexus class-code` | Get full class code with methods/fields |
| `search_symbols` | `gitnexus search` | Fuzzy search symbols by name or keyword |
| `get_file_symbols` | `gitnexus file-symbols` | Get all symbols in a file |

---

## Tool 1: `find_implementations`

### Purpose
Find all classes that implement a given interface or extend a base class.

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `interface_name` | string | Yes | - | Interface or base class name |
| `fuzzy_match` | boolean | No | true | Enable fuzzy matching (CONTAINS) |
| `include_content` | boolean | No | false | Return implementation code |
| `repo` | string | No | - | Repository name or path |

### CLI Usage

```bash
# Exact match
gitnexus find-impl --interface "IUserService"

# Fuzzy match (default)
gitnexus find-impl --interface "UserService" --fuzzy

# With code content
gitnexus find-impl --interface "IUserService" --content
```

### MCP Usage

```json
{
  "name": "find_implementations",
  "arguments": {
    "interface_name": "IUserService",
    "fuzzy_match": true,
    "include_content": false
  }
}
```

### Return Structure

```json
{
  "interface": "IUserService",
  "implementations": [
    {
      "name": "UserServiceImpl",
      "type": "Class",
      "filePath": "src/services/UserServiceImpl.java",
      "startLine": 15,
      "endLine": 120,
      "content": "...",
      "methods": ["getUser", "createUser", "updateUser"]
    }
  ],
  "total": 1
}
```

### Cypher Query

```cypher
// Exact match
MATCH (impl)-[r:CodeRelation {type: 'IMPLEMENTS'}]->(iface {name: $name})
RETURN impl.name, impl.filePath, impl.startLine, impl.endLine

// Fuzzy match
MATCH (impl)-[r:CodeRelation {type: 'IMPLEMENTS'}]->(iface)
WHERE iface.name CONTAINS $name
RETURN impl.name, impl.filePath, iface.name AS interfaceName
```

---

## Tool 2: `get_class_code`

### Purpose
Get complete class code including methods, fields, and relationships.

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `class_name` | string | Yes* | - | Class name (supports fuzzy) |
| `file_path` | string | No | - | File path for exact match |
| `include_methods` | boolean | No | true | Include method list |
| `include_fields` | boolean | No | true | Include field list |
| `include_content` | boolean | No | true | Return full code |
| `repo` | string | No | - | Repository name or path |

*Either `class_name` or `file_path` is required.

### CLI Usage

```bash
# By class name
gitnexus class-code --name "UserService"

# By file path
gitnexus class-code --file "src/services/UserService.java"

# Without method details
gitnexus class-code --name "UserService" --no-methods

# Without code content (metadata only)
gitnexus class-code --name "UserService" --no-content
```

### MCP Usage

```json
{
  "name": "get_class_code",
  "arguments": {
    "class_name": "UserService",
    "include_methods": true,
    "include_fields": true,
    "include_content": true
  }
}
```

### Return Structure

```json
{
  "class": {
    "name": "UserService",
    "type": "Class",
    "filePath": "src/services/UserService.java",
    "startLine": 10,
    "endLine": 150,
    "content": "public class UserService {\n  ...\n}",
    "isExported": true
  },
  "methods": [
    {
      "name": "getUser",
      "startLine": 25,
      "endLine": 40,
      "parameterCount": 1,
      "returnType": "User",
      "content": "public User getUser(String id) {...}"
    }
  ],
  "fields": [
    {
      "name": "userRepository",
      "type": "UserRepository",
      "startLine": 12
    }
  ],
  "implements": ["IUserService", "Serializable"],
  "extends": "BaseService",
  "incoming": {
    "callers": ["UserController", "AuthController"],
    "importers": ["src/controllers/UserController.java"]
  }
}
```

### Cypher Query

```cypher
// Find class
MATCH (c:Class) WHERE c.name = $name OR c.name CONTAINS $name
RETURN c

// Get methods
MATCH (c:Class {id: $classId})-[r:CodeRelation {type: 'HAS_METHOD'}]->(m:Method)
RETURN m.name, m.startLine, m.endLine, m.parameterCount, m.returnType

// Get fields
MATCH (c:Class {id: $classId})-[r:CodeRelation {type: 'HAS_PROPERTY'}]->(p:Property)
RETURN p.name, p.declaredType

// Get implements
MATCH (c:Class {id: $classId})-[r:CodeRelation {type: 'IMPLEMENTS'}]->(i:Interface)
RETURN i.name

// Get extends
MATCH (c:Class {id: $classId})-[r:CodeRelation {type: 'EXTENDS'}]->(parent)
RETURN parent.name
```

---

## Tool 3: `search_symbols`

### Purpose
Fuzzy search symbols by name or keyword in code content.

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `query` | string | Yes | - | Search keyword or code snippet |
| `symbol_type` | string | No | "all" | Filter: Class, Interface, Method, Function, all |
| `file_path` | string | No | - | Filter by file path |
| `fuzzy_match` | boolean | No | true | Enable fuzzy matching |
| `search_content` | boolean | No | false | Search in code content (not just name) |
| `include_content` | boolean | No | false | Return code content |
| `limit` | number | No | 20 | Max results |
| `repo` | string | No | - | Repository name or path |

### CLI Usage

```bash
# Search by name
gitnexus search --query "getUser"

# Search in specific type
gitnexus search --query "Service" --type Class

# Search in code content
gitnexus search --query "userRepository.save" --content-search

# Filter by file
gitnexus search --query "getUser" --file "src/services"

# With code content
gitnexus search --query "getUser" --content
```

### MCP Usage

```json
{
  "name": "search_symbols",
  "arguments": {
    "query": "getUser",
    "symbol_type": "Method",
    "search_content": false,
    "include_content": true,
    "limit": 10
  }
}
```

### Return Structure

```json
{
  "query": "getUser",
  "results": [
    {
      "name": "getUser",
      "type": "Method",
      "filePath": "src/services/UserService.java",
      "startLine": 25,
      "endLine": 40,
      "content": "public User getUser(String id) {...}",
      "parent": {
        "name": "UserService",
        "type": "Class"
      },
      "matchType": "exact"
    }
  ],
  "total": 1
}
```

### Cypher Query

```cypher
// Search by name (exact)
MATCH (n) WHERE labels(n) IN $types AND n.name = $query
RETURN n

// Search by name (fuzzy)
MATCH (n) WHERE labels(n) IN $types AND n.name CONTAINS $query
RETURN n

// Search in content
MATCH (n) WHERE labels(n) IN $types AND n.content CONTAINS $query
RETURN n
```

---

## Tool 4: `get_file_symbols`

### Purpose
Get all symbols (classes, methods, functions) in a file by file path.

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `file_path` | string | Yes | - | File path (supports partial match) |
| `include_content` | boolean | No | false | Return code content for each symbol |
| `symbol_type` | string | No | "all" | Filter: Class, Interface, Method, Function, all |
| `repo` | string | No | - | Repository name or path |

### CLI Usage

```bash
# Get all symbols in a file
gitnexus file-symbols --path "src/services/UserService.java"

# Partial path match
gitnexus file-symbols --path "UserService.java"

# Filter by type
gitnexus file-symbols --path "UserService.java" --type Class

# With code content
gitnexus file-symbols --path "UserService.java" --content
```

### MCP Usage

```json
{
  "name": "get_file_symbols",
  "arguments": {
    "file_path": "src/services/UserService.java",
    "include_content": true,
    "symbol_type": "all"
  }
}
```

### Return Structure

```json
{
  "file": {
    "filePath": "src/services/UserService.java",
    "name": "UserService.java"
  },
  "symbols": [
    {
      "name": "UserService",
      "type": "Class",
      "startLine": 10,
      "endLine": 150,
      "content": "public class UserService {...}",
      "isExported": true
    },
    {
      "name": "getUser",
      "type": "Method",
      "startLine": 25,
      "endLine": 40,
      "content": "public User getUser(String id) {...}",
      "parent": "UserService"
    }
  ],
  "total": 2
}
```

### Cypher Query

```cypher
// Get file
MATCH (f:File) WHERE f.filePath CONTAINS $path OR f.name = $path
RETURN f

// Get symbols in file
MATCH (f:File)-[r:CodeRelation {type: 'DEFINES'}]->(n)
WHERE f.filePath CONTAINS $path AND labels(n)[0] IN $types
RETURN n.name, labels(n)[0] AS type, n.startLine, n.endLine, n.content
ORDER BY n.startLine
```

---

## Implementation Plan

### Files to Modify

| File | Changes |
|------|---------|
| `gitnexus/src/mcp/tools.ts` | Add 4 tool definitions |
| `gitnexus/src/mcp/local/local-backend.ts` | Add 4 private methods |
| `gitnexus/src/mcp/server.ts` | Register 4 tools |
| `gitnexus/src/cli/tool.ts` | Add 4 CLI command functions |
| `gitnexus/src/cli/index.ts` | Register 4 CLI commands |

### Implementation Order

1. **Phase 1: Core Implementation**
   - Add tool definitions to `tools.ts`
   - Implement backend methods in `local-backend.ts`
   - Register tools in `server.ts`

2. **Phase 2: CLI Support**
   - Add CLI command functions to `tool.ts`
   - Register commands in `index.ts`

3. **Phase 3: Testing**
   - Unit tests for each tool
   - Integration tests for CLI commands

---

## Out of Scope

- Web UI support (can be added later)
- Streaming responses
- Caching layer (can be added if performance is an issue)

---

## Success Criteria

1. `gitnexus find-impl --interface "IUserService"` returns all implementing classes
2. `gitnexus class-code --name "UserService"` returns complete class code
3. `gitnexus search --query "getUser" --type Method` finds all matching methods
4. `gitnexus file-symbols --path "UserService.java"` returns all symbols in the file
5. All 4 tools work via MCP for AI agents