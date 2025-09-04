# Cursorless MCP: Product Requirements Document

## Overview

This document outlines the requirements and design for building a Model Context Protocol (MCP) server that exposes Cursorless's syntax tree manipulation capabilities to Large Language Models (LLMs). The MCP will enable LLMs to interact with code at the semantic level using Cursorless's sophisticated targeting and action system, moving beyond text-based manipulation to true structural code editing.

## Project Goals

- **Enable LLMs to perform semantic code manipulation** through Cursorless's proven targeting system
- **Provide syntax-aware code editing capabilities** that understand language structure and semantics
- **Bridge the gap between natural language requests and precise code transformations**
- **Leverage Cursorless's extensive language support** (40+ programming languages via tree-sitter)
- **Support complex multi-step code transformations** through action composition

## 1. Overview of Cursorless's Underlying Interface

### 1.1 Core Architecture

Cursorless is built around a modular engine that processes commands to manipulate code based on its syntactic structure. The system consists of several key components:

#### Engine Architecture
```
Command Input → Engine → Target Pipeline → Actions → Code Transformation
     ↓              ↓           ↓             ↓            ↓
  Command       Target      Scope       Action        Updated
 Descriptor   Processing   Handlers    Execution       Code
```

### 1.2 Command System

The core interface revolves around the `Command` structure ([CommandV6.types.ts](packages/common/src/types/command/CommandV6.types.ts)):

```typescript
interface Command {
  version: number;                    // API version (currently 7)
  spokenForm?: string;               // Original voice command (optional)
  usePrePhraseSnapshot: boolean;     // For voice integration
  action: ActionDescriptor;          // What to do
}
```

#### Action Types
Cursorless supports 60+ action types organized into categories ([ActionDescriptor.ts](packages/common/src/types/command/ActionDescriptor.ts)):

**Simple Actions** (single target operations):
- `setSelection`, `remove`, `copy`, `cut`
- `editNewLineBefore`, `editNewLineAfter`
- `increment`, `decrement`, `rename`
- `scrollToTop`, `foldRegion`, `showHover`

**Complex Actions** (multi-target or parameterized):
- `replaceWithTarget`, `moveToTarget` (bring/move operations)
- `swapTargets`, `callAsFunction`
- `wrapWithPairedDelimiter`, `insertSnippet`
- `executeCommand`, `getText`

#### Action Descriptor Types

| Action Descriptor Type | Description | Required Parameters | Optional Parameters | Code Reference |
|------------------------|-------------|--------------------|--------------------|----------------|
| `SimpleActionDescriptor` | Single target operations (66 actions) | `name: SimpleActionName`<br/>`target: PartialTargetDescriptor` | None | [Lines 98-101](packages/common/src/types/command/ActionDescriptor.ts#L98-L101) |
| `BringMoveActionDescriptor` | Move/replace operations between targets | `name: "replaceWithTarget" \| "moveToTarget"`<br/>`source: PartialTargetDescriptor`<br/>`destination: DestinationDescriptor` | None | [Lines 103-107](packages/common/src/types/command/ActionDescriptor.ts#L103-L107) |
| `SwapActionDescriptor` | Swap two targets | `name: "swapTargets"`<br/>`target1: PartialTargetDescriptor`<br/>`target2: PartialTargetDescriptor` | None | [Lines 123-127](packages/common/src/types/command/ActionDescriptor.ts#L123-L127) |
| `CallActionDescriptor` | Wrap target in function call | `name: "callAsFunction"`<br/>`callee: PartialTargetDescriptor`<br/>`argument: PartialTargetDescriptor` | None | [Lines 109-121](packages/common/src/types/command/ActionDescriptor.ts#L109-L121) |
| `PasteActionDescriptor` | Paste from clipboard to destination | `name: "pasteFromClipboard"`<br/>`destination: DestinationDescriptor` | None | [Lines 136-139](packages/common/src/types/command/ActionDescriptor.ts#L136-L139) |
| `ExecuteCommandActionDescriptor` | Execute IDE commands on targets | `name: "executeCommand"`<br/>`commandId: string`<br/>`target: PartialTargetDescriptor` | `options?: ExecuteCommandOptions`<br/>- `commandArgs?: any[]`<br/>- `ensureSingleEditor?: boolean`<br/>- `ensureSingleTarget?: boolean`<br/>- `restoreSelection?: boolean`<br/>- `showDecorations?: boolean` | [Lines 219-224](packages/common/src/types/command/ActionDescriptor.ts#L219-L224) |
| `ReplaceActionDescriptor` | Replace target with string/pattern | `name: "replace"`<br/>`replaceWith: string[] \| { start: number }`<br/>`destination: DestinationDescriptor` | None | [Lines 228-232](packages/common/src/types/command/ActionDescriptor.ts#L228-L232) |
| `HighlightActionDescriptor` | Highlight targets | `name: "highlight"`<br/>`target: PartialTargetDescriptor` | `highlightId?: string` | [Lines 234-238](packages/common/src/types/command/ActionDescriptor.ts#L234-L238) |
| `GenerateSnippetActionDescriptor` | Generate snippet from target | `name: "generateSnippet"`<br/>`target: PartialTargetDescriptor` | `directory?: string`<br/>`snippetName?: string` | [Lines 141-146](packages/common/src/types/command/ActionDescriptor.ts#L141-L146) |
| `InsertSnippetActionDescriptor` | Insert snippet at destination | `name: "insertSnippet"`<br/>`snippetDescription: InsertSnippetArg`<br/>`destination: DestinationDescriptor` | None (but `InsertSnippetArg` has optional fields) | [Lines 174-178](packages/common/src/types/command/ActionDescriptor.ts#L174-L178) |
| `WrapWithSnippetActionDescriptor` | Wrap target with snippet | `name: "wrapWithSnippet"`<br/>`snippetDescription: WrapWithSnippetArg`<br/>`target: PartialTargetDescriptor` | None (but `WrapWithSnippetArg` has optional fields) | [Lines 205-209](packages/common/src/types/command/ActionDescriptor.ts#L205-L209) |
| `WrapWithPairedDelimiterActionDescriptor` | Wrap target with delimiters | `name: "wrapWithPairedDelimiter" \| "rewrapWithPairedDelimiter"`<br/>`left: string`<br/>`right: string`<br/>`target: PartialTargetDescriptor` | None | [Lines 129-134](packages/common/src/types/command/ActionDescriptor.ts#L129-L134) |
| `EditNewActionDescriptor` | Create new editor at destination | `name: "editNew"`<br/>`destination: DestinationDescriptor` | None | [Lines 240-243](packages/common/src/types/command/ActionDescriptor.ts#L240-L243) |
| `GetTextActionDescriptor` | Extract text from targets | `name: "getText"`<br/>`target: PartialTargetDescriptor` | `options?: GetTextActionOptions`<br/>- `showDecorations?: boolean`<br/>- `ensureSingleTarget?: boolean` | [Lines 250-254](packages/common/src/types/command/ActionDescriptor.ts#L250-L254) |
| `ParsedActionDescriptor` | Parsed command actions | `name: "parsed"`<br/>`content: string`<br/>`arguments: unknown[]` | None | [Lines 256-260](packages/common/src/types/command/ActionDescriptor.ts#L256-L260) |

### 1.3 Target System

The target system is Cursorless's most sophisticated component, enabling precise selection of code elements:

#### Target Descriptors
([PartialTargetDescriptor.types.ts](packages/common/src/types/command/PartialTargetDescriptor.types.ts#L522-527))
```typescript
type PartialTargetDescriptor =
  | PartialPrimitiveTargetDescriptor    // Single element
  | PartialRangeTargetDescriptor        // Range between two elements
  | PartialListTargetDescriptor         // Multiple discrete elements
  | ImplicitTargetDescriptor            // Context-inferred target
```

#### Marks (Starting Points)
- **Decorated symbols**: `{color: "red", character: "a"}` - Hat-decorated tokens
- **Cursor positions**: `"cursor"`, `"that"` (previous target), `"source"`
- **Line numbers**: Absolute, relative, or modulo-100
- **Explicit ranges**: Direct coordinate specification

#### Scope Types (What to Target)
Cursorless defines 80+ scope types across multiple categories:

**Programming Constructs** ([PartialTargetDescriptor.types.ts](packages/common/src/types/command/PartialTargetDescriptor.types.ts#142), [spoken_forms.json](cursorless-talon/src/spoken_forms.json)):
- `namedFunction`, `anonymousFunction`, `functionCall`, `functionName`
- `class`, `className`, `ifStatement`, `statement`
- `argumentOrParameter`, `argumentList`, `value`, `type`

**Text-Based Scopes**:
- `word`, `token`, `identifier`, `line`, `paragraph`
- `string`, `comment`, `url`

**Structural Elements**:
- `list`, `map`, `collectionItem`, `collectionKey`
- Various surrounding pairs: `parentheses`, `curlyBrackets`, `doubleQuotes`

#### Modifiers (How to Transform Targets)
- **Scope Navigation**: `containingScope`, `everyScope`
- **Relative Movement**: `next`, `previous`, `first`, `last`
- **Range Construction**: `head`, `tail`, `inside`
- **Filtering**: `visible`, `content`, `empty`

### 1.4 Language Support System

#### Tree-sitter Integration
Cursorless uses tree-sitter for parsing, with language-specific query files ([queries/](queries/)):

```scheme
;; Example from javascript.scm
(function_declaration
  name: (identifier) @name @functionName
  parameters: (formal_parameters) @argumentList
  body: (statement_block) @_.interior
) @namedFunction @statement
```

#### Language Definitions
- **40+ supported languages** with dedicated query files
- **Scope handlers** that map AST nodes to semantic concepts
- **Dynamic loading** based on active editor language
- **Extensible system** for adding new languages

#### Scope Provider Interface
([ScopeProvider.ts](packages/common/src/types/ScopeProvider.ts))
```typescript
interface ScopeProvider {
  provideScopeRanges(editor: TextEditor, config: ScopeRangeConfig): ScopeRanges[];
  getScopeSupport(editor: TextEditor, scopeType: ScopeType): ScopeSupport;
  onDidChangeScopeRanges(callback: ScopeChangeEventCallback): Disposable;
}
```

### 1.5 Engine API Surface

#### Core Engine Interface
([CursorlessEngineApi.ts](packages/cursorless-engine/src/api/CursorlessEngineApi.ts))
```typescript
interface CursorlessEngine {
  commandApi: CommandApi;                      // Command execution
  scopeProvider: ScopeProvider;               // Scope analysis
  customSpokenFormGenerator: CustomSpokenFormGenerator; // Voice integration
  storedTargets: StoredTargetMap;             // Target persistence
  hatTokenMap: HatTokenMap;                   // Decoration system
}
```

#### Command Execution
([CursorlessEngineApi.ts](packages/cursorless-engine/src/api/CursorlessEngineApi.ts))
```typescript
interface CommandApi {
  runCommand(command: Command): Promise<CommandResponse>;
  runCommandSafe(...args: unknown[]): Promise<CommandResponse>;
  repeatPreviousCommand(): Promise<CommandResponse>;
}
```

#### IDE Abstraction
The engine operates through an IDE abstraction layer that provides:
- **Text document access and manipulation**
- **Selection and cursor management**
- **Language service integration**
- **File system operations**
- **Command palette integration**

### 1.6 Data Flow

1. **Command Reception**: Commands arrive as structured objects or raw arguments
2. **Canonicalization**: Commands are validated and upgraded to current version ([runCommand.ts](packages/cursorless-engine/src/runCommand.ts))
3. **Target Resolution**: Target descriptors are processed through the pipeline:
   - **Mark Stage**: Resolve starting positions/ranges
   - **Modifier Stage**: Apply scope transformations and filtering
   - **Output**: Concrete editor ranges with semantic metadata
4. **Action Execution**: Actions operate on resolved targets
5. **Result**: Code is modified and response returned

### 1.7 Key Capabilities for MCP Integration

#### Precise Code Targeting
- **Semantic awareness**: Target functions, classes, arguments by meaning
- **Language-agnostic interface**: Same commands work across programming languages
- **Complex selections**: Ranges, lists, nested scopes, relative navigation

#### Rich Action Set
- **CRUD operations**: Create, read, update, delete code elements
- **Structural transformations**: Move, swap, wrap, extract operations
- **Navigation and inspection**: Jump to definitions, show documentation
- **Snippet integration**: Template-based code generation

#### Extensibility
- **Custom actions**: Plugin system for domain-specific operations
- **Language support**: Add new languages via tree-sitter queries
- **Scope definitions**: Define new semantic concepts

This foundation provides everything needed for an MCP that can:
- Accept high-level semantic requests from LLMs
- Translate them into precise Cursorless commands
- Execute sophisticated code transformations
- Return structured results with semantic metadata

The next sections will detail how to build the MCP layer on top of this powerful foundation.

## 2. Building with MCP: Key Components and Architecture

### 2.1 Model Context Protocol Overview

The Model Context Protocol (MCP) is an open standard that enables AI applications to securely connect to external data sources and tools, providing structured context to Large Language Models. MCP acts as a universal adapter—similar to how USB-C standardizes device connections—allowing AI applications to access data and functionality without requiring custom integrations for each service.

For the Cursorless MCP server, this means LLMs can access Cursorless's sophisticated code manipulation capabilities through a standardized interface, enabling semantic code editing without the complexity of directly interfacing with the underlying engine.

### 2.2 Core MCP Components

MCP defines three fundamental component types that will form the backbone of our Cursorless integration:

**Resources** serve as read-only endpoints that expose data without side effects. In our Cursorless context, resources will provide:
- Current code structure and syntax tree information
- Available scope types and their definitions for different languages
- Target analysis results (what elements can be targeted in the current code)
- Language capability metadata (supported scopes, actions per language)

**Tools** function as write endpoints that allow LLMs to perform actions and execute code transformations. These will map directly to Cursorless's action system:
- Code manipulation tools (remove, replace, move, swap, wrap)
- Navigation and selection tools (setSelection, scrollTo, highlight)
- Structural transformation tools (extract function, rename, refactor)
- Snippet and template insertion tools

**Prompts** provide reusable templates that define interaction patterns, helping LLMs understand how to effectively use Cursorless capabilities:
- Common code transformation patterns
- Language-specific operation templates
- Multi-step workflow guides for complex refactoring

### 2.3 TypeScript SDK Implementation Strategy

The official MCP TypeScript SDK provides the foundation for our server implementation. The `McpServer` class will serve as our core interface, handling protocol compliance and message routing between LLMs and the Cursorless engine.

Our server architecture will initialize with metadata about Cursorless's capabilities:

```typescript
const cursorlessServer = new McpServer({
  name: "Cursorless Semantic Code Editor",
  version: "1.0.0",
  description: "Syntax-aware code manipulation through semantic targeting"
});
```

Resource definitions will expose Cursorless's analysis capabilities through both static and dynamic endpoints. Static resources will provide language definitions and scope hierarchies, while dynamic resources will analyze specific code files and provide real-time syntax tree information parameterized by file path and target queries.

Tool implementations will create a direct bridge between MCP tool calls and Cursorless commands, translating LLM requests into the structured `Command` objects that the Cursorless engine expects. Each tool will encapsulate the command construction logic, target resolution, and result formatting needed to make Cursorless's powerful targeting system accessible to LLMs.

The modular design of both MCP and Cursorless creates a natural architectural alignment—MCP's resource/tool separation maps cleanly onto Cursorless's read/write operation distinction, while MCP's standardized communication protocol provides the structure needed to expose Cursorless's sophisticated targeting system to AI applications.

### 2.4 Example Transformations

To illustrate how LLMs will interact with Cursorless through MCP, here are two common code transformation scenarios:
**NOTE:** The examples below are a thought experiment and not the actual MCP interface, as the targeting system is unrealistic. See section 3 for more discussion on that topic

#### Example 1: Adding an Argument to a Function

**LLM Request**: "Add a `timeout` parameter of type `number` with default value `5000` to the `fetchData` function"

**MCP Tool Call**:
```json
{
  "tool": "cursorless_transform",
  "arguments": {
    "action": "insertSnippet",
    "target": {
      "type": "primitive",
      "mark": {"type": "decoratedSymbol", "color": "red", "character": "a"},
      "modifiers": [
        {"type": "containingScope", "scopeType": "argumentList"}
      ]
    },
    "destination": {
      "type": "after",
      "target": "that"
    },
    "snippet": {
      "name": "parameter",
      "body": ", timeout: number = 5000"
    }
  }
}
```

**Cursorless Command Generated**:
```typescript
{
  version: 7,
  usePrePhraseSnapshot: false,
  action: {
    name: "insertSnippet",
    snippetDescription: {
      type: "named",
      name: "parameter",
      substitutions: {
        "paramName": "timeout",
        "paramType": "number",
        "defaultValue": "5000"
      }
    } as NamedInsertSnippetArg,
    destination: {
      type: "primitive",
      insertionMode: "after",
      target: {
        type: "primitive",
        mark: {
          type: "decoratedSymbol",
          symbolColor: "red",
          character: "a"
        } as DecoratedSymbolMark,
        modifiers: [
          { type: "containingScope", scopeType: "argumentList" },
          { type: "last" }
        ]
      } as PartialPrimitiveTargetDescriptor
    } as PrimitiveDestinationDescriptor
  } as InsertSnippetActionDescriptor
}
```

**Code Transformation**:
```javascript
// Before
function fetchData(url) {
  return fetch(url);
}

// After
function fetchData(url, timeout: number = 5000) {
  return fetch(url);
}
```

#### Example 2: Adding a Logical Branch to an If Statement

**LLM Request**: "Add an else if branch to check if status is 'pending'"

**MCP Tool Call**:
```json
{
  "tool": "cursorless_transform",
  "arguments": {
    "action": "insertSnippet",
    "target": {
      "type": "primitive",
      "mark": {"type": "decoratedSymbol", "color": "blue", "character": "b"},
      "modifiers": [
        {"type": "containingScope", "scopeType": "ifStatement"}
      ]
    },
    "destination": {
      "type": "after",
      "target": "that"
    },
    "snippet": {
      "name": "elseIfBranch",
      "body": " else if (status === 'pending') {\n  // Handle pending state\n}"
    }
  }
}
```

**Cursorless Command Generated**:
```typescript
{
  version: 7,
  usePrePhraseSnapshot: false,
  action: {
    name: "insertSnippet",
    snippetDescription: {
      type: "named",
      name: "elseIfBranch",
      substitutions: {
        "condition": "status === 'pending'",
        "comment": "Handle pending state"
      }
    } as NamedInsertSnippetArg,
    destination: {
      type: "primitive",
      insertionMode: "after",
      target: {
        type: "primitive",
        mark: {
          type: "decoratedSymbol",
          symbolColor: "blue",
          character: "b"
        } as DecoratedSymbolMark,
        modifiers: [
          { type: "containingScope", scopeType: "ifStatement" }
        ]
      } as PartialPrimitiveTargetDescriptor
    } as PrimitiveDestinationDescriptor
  } as InsertSnippetActionDescriptor
}
```

**Code Transformation**:
```javascript
// Before
if (status === 'success') {
  return data;
}

// After
if (status === 'success') {
  return data;
} else if (status === 'pending') {
  // Handle pending state
}
```

#### Type References

The commands above use the following precise Cursorless type definitions:

- [`InsertSnippetActionDescriptor`](packages/common/src/types/command/ActionDescriptor.ts#L174-L178) - Action for inserting snippets at destinations
- [`NamedInsertSnippetArg`](packages/common/src/types/command/ActionDescriptor.ts#L148-L152) - Snippet argument with name and substitutions
- [`PrimitiveDestinationDescriptor`](packages/common/src/types/command/DestinationDescriptor.types.ts#L21-L33) - Single insertion destination with mode
- [`PartialPrimitiveTargetDescriptor`](packages/common/src/types/command/PartialTargetDescriptor.types.ts#L498-L502) - Single target with mark and modifiers
- [`DecoratedSymbolMark`](packages/common/src/types/command/PartialTargetDescriptor.types.ts#L32-L36) - Hat-decorated symbol reference
- [`InsertionMode`](packages/common/src/types/command/DestinationDescriptor.types.ts#L19) - `"before" | "after" | "to"` positioning modes

These examples demonstrate how the MCP layer abstracts away Cursorless's complexity while preserving its semantic precision. The LLM provides high-level intent, the MCP translates this into structured tool calls, and Cursorless executes the precise syntactic transformations using its proven targeting system.

## 3. LLM Targeting: Interface Design for Code Manipulation

### 3.1 The Challenge of LLM-to-Code Interfaces

The fundamental challenge in building an effective LLM interface for Cursorless lies in bridging the gap between how LLMs perceive code (as text) and how Cursorless operates on it (as semantic structures). The decorated symbols used in the MCP examples above—visual hat decorations like `{color: "red", character: "a"}`—represent a significant limitation: these visual affordances exist only in the editor interface and are not accessible to LLMs working with raw code text.

This section explores established techniques for LLM code editing and proposes a robust targeting interface that enables semantic code manipulation without relying on visual decorations.

### 3.2 Current Approaches to AI Code Editing

Recent analysis of AI coding assistants reveals several sophisticated approaches to code manipulation, each with distinct advantages and limitations. Understanding these techniques is crucial for designing an effective LLM interface for Cursorless.

#### 3.2.1 Patch-Based Systems

**OpenAI's Codex Approach** ([fabianhertwig.com](https://fabianhertwig.com/blog/coding-assistants-file-edits/)) demonstrates a structured patch format that avoids line number dependencies:

```
*** Begin Patch
*** Update File: main.py
@@ def main():
   # This is the main function
-  print("hello")
+  print("hello world!")
   return None
*** End Patch
```

**Key Principles:**
- **Context-based targeting**: Uses surrounding code lines for precise location
- **Content-focused matching**: Avoids fragile line numbers
- **Progressive fallback**: Exact match → trimmed whitespace → all whitespace ignored
- **Semantic anchoring**: `@@` lines reference nearby function/class definitions

**Limitations for Cursorless Integration:**
- Operates at text level rather than semantic structures
- Limited to simple find-and-replace operations
- Cannot leverage Cursorless's sophisticated scope targeting
- Requires exact text matching, fragile to minor variations

#### 3.2.2 Search/Replace Block Systems

**Aider and RooCode** implement intuitive delimiter-based formats:

```
<<<<<<< SEARCH
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item, 0);
}
=======
function calculateTotal(items) {
  // Add 10% tax
  return items.reduce((sum, item) => sum + (item * 1.1), 0);
}
>>>>>>> REPLACE
```

**Advanced Features (RooCode):**
- **Middle-out fuzzy matching**: Searches outward from estimated locations
- **Levenshtein distance scoring**: Handles minor variations in code
- **Sophisticated indentation preservation**: Maintains whitespace formatting
- **Multi-part edits**: Single command can modify multiple file sections

**Benefits:**
- Intuitive for LLMs to generate
- Robust against minor formatting differences
- Explicit about intended changes
- Good error reporting for debugging

**Limitations:**
- Still text-based rather than semantic
- Cannot target by structural meaning (e.g., "the second parameter")
- Limited to contiguous code blocks
- No understanding of language-specific constructs

#### 3.2.3 AI-Assisted Application

**Cursor's Two-Model Approach** represents a significant advancement in edit reliability:

1. **Sketching Model**: Generates intended change with focus on logic rather than perfect syntax
2. **Application Model**: Specialized AI trained to integrate changes into existing codebases

**Advantages:**
- Separates high-level intent from low-level integration
- Custom training for edit application tasks
- Better handling of complex, context-dependent changes
- More robust than algorithmic approaches

**Relevance to Cursorless:**
- Demonstrates value of specialized models for code integration
- Shows promise of AI-assisted targeting and application
- Could complement Cursorless's semantic understanding with LLM flexibility

### 3.3 Proposed LLM Interface for Cursorless

Building on these established approaches while leveraging Cursorless's unique semantic capabilities, we propose a multi-layered targeting interface that eliminates dependency on visual decorations.

#### 3.3.1 Semantic Query Language

Instead of relying on decorated symbols, LLMs will specify targets using a semantic query language that maps directly to Cursorless's targeting system:

```typescript
interface SemanticTargetDescriptor {
  // Primary targeting strategy
  scope: ScopeType;                    // "function", "parameter", "ifStatement"

  // Content-based identification
  containing?: string;                 // Text that should appear within the target
  matching?: string | RegExp;          // Pattern the target content should match

  // Structural context
  within?: SemanticTargetDescriptor;   // Parent scope for disambiguation
  position?: "first" | "last" | number; // Position among siblings

  // Relative positioning
  relative?: {
    to: SemanticTargetDescriptor;
    direction: "before" | "after" | "inside";
  };
}
```

**Example Usage:**
```json
{
  "scope": "parameter",
  "containing": "timeout",
  "within": {
    "scope": "function",
    "containing": "fetchData"
  }
}
```

This targets "the parameter containing 'timeout' within the function containing 'fetchData'".

#### 3.3.2 Content-Aware Fuzzy Matching

To handle the reality that LLMs may not have exact code text, implement progressive matching strategies:

**Level 1: Exact Content Match**
```typescript
{
  "scope": "functionCall",
  "matching": "fetch(url, { timeout: 5000 })"
}
```

**Level 2: Fuzzy Content Match**
```typescript
{
  "scope": "functionCall",
  "containing": "fetch",
  "parameters": ["url", "timeout"]
}
```

**Level 3: Structural Pattern Match**
```typescript
{
  "scope": "functionCall",
  "pattern": {
    "name": "fetch",
    "argumentCount": 2,
    "firstArgumentType": "identifier"
  }
}
```

#### 3.3.3 Multi-Strategy Target Resolution

Implement a cascading resolution system that tries multiple approaches:

1. **Semantic + Content**: Primary approach using scope type and content matching
2. **Pattern + Context**: Fallback using structural patterns within context
3. **Fuzzy + Heuristics**: Final fallback using similarity scoring and code analysis
4. **Interactive Disambiguation**: When multiple matches exist, provide options to LLM

```typescript
interface TargetResolutionResult {
  success: boolean;
  targets: ResolvedTarget[];
  confidence: number;
  alternatives?: ResolvedTarget[];
  diagnostics: TargetDiagnostic[];
}
```

#### 3.3.4 Context-Aware Targeting Tools

Define MCP tools that leverage both semantic understanding and content analysis:

**Tool: `analyze_code_structure`**
```typescript
{
  "name": "analyze_code_structure",
  "description": "Analyze code to identify targetable elements",
  "inputSchema": {
    "type": "object",
    "properties": {
      "file": {"type": "string"},
      "scopeTypes": {"type": "array", "items": {"type": "string"}},
      "region": {"type": "object"} // Optional focus area
    }
  }
}
```

**Tool: `target_code_element`**
```typescript
{
  "name": "target_code_element",
  "description": "Select code elements using semantic queries",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {"$ref": "#/definitions/SemanticTargetDescriptor"},
      "action": {"type": "string"},
      "parameters": {"type": "object"}
    }
  }
}
```

**Tool: `preview_targeting`**
```typescript
{
  "name": "preview_targeting",
  "description": "Preview what would be targeted without making changes",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {"$ref": "#/definitions/SemanticTargetDescriptor"},
      "showAlternatives": {"type": "boolean", "default": true}
    }
  }
}
```

#### 3.3.5 Robust Error Handling and Feedback

Learning from successful systems like Aider, provide detailed diagnostic information:

```typescript
interface TargetingDiagnostic {
  level: "error" | "warning" | "info";
  message: string;
  suggestion?: string;
  context?: {
    file: string;
    line: number;
    column: number;
    surrounding: string[];
  };
  alternatives?: SemanticTargetDescriptor[];
}
```

**Example Error Response:**
```json
{
  "success": false,
  "error": "Multiple functions containing 'fetchData' found",
  "diagnostics": [
    {
      "level": "error",
      "message": "Ambiguous target: 3 functions match the criteria",
      "alternatives": [
        {
          "scope": "function",
          "containing": "fetchData",
          "within": {"scope": "class", "containing": "ApiClient"}
        },
        {
          "scope": "function",
          "containing": "fetchData",
          "within": {"scope": "module", "containing": "utils"}
        }
      ]
    }
  ]
}
```

### 3.4 Implementation Strategy

#### 3.4.1 Progressive Enhancement Approach

**Phase 1: Basic Semantic Targeting**
- Implement core semantic query language
- Support primary scope types and content matching
- Basic error handling and feedback

**Phase 2: Advanced Pattern Matching**
- Add fuzzy matching capabilities
- Implement structural pattern recognition
- Enhanced disambiguation strategies

**Phase 3: AI-Assisted Targeting**
- Integrate LLM-based target resolution for complex cases
- Implement learning from user corrections
- Advanced context understanding

#### 3.4.2 Integration with Cursorless Engine

The semantic targeting layer will translate LLM queries into native Cursorless commands:

```typescript
class SemanticTargetResolver {
  constructor(private cursorlessEngine: CursorlessEngine) {}

  async resolveTarget(
    query: SemanticTargetDescriptor,
    context: FileContext
  ): Promise<PartialTargetDescriptor> {
    // 1. Analyze file structure using Cursorless scope providers
    const scopes = await this.analyzeScopeStructure(context);

    // 2. Apply content-based filtering
    const candidates = this.filterByContent(scopes, query);

    // 3. Apply positional constraints
    const positioned = this.applyPositioning(candidates, query);

    // 4. Convert to Cursorless target descriptor
    return this.toCursorlessTarget(positioned);
  }
}
```

#### 3.4.3 Testing and Validation

**Comprehensive Test Suite:**
- Unit tests for semantic query parsing
- Integration tests with various code patterns
- Edge case handling (ambiguous targets, malformed queries)
- Performance tests with large codebases

**Real-World Validation:**
- Test with actual LLM-generated queries
- Measure targeting accuracy across different coding scenarios
- Collect feedback on error message clarity and helpfulness

### 3.5 Benefits of the Proposed Approach

**For LLMs:**
- Natural, text-based interface without visual dependencies
- Rich semantic vocabulary for precise targeting
- Robust error handling enables iterative refinement
- Progressive fallback ensures high success rates

**For Developers:**
- Maintains Cursorless's semantic precision
- Leverages existing scope provider infrastructure
- Extensible to new languages and scope types
- Compatible with existing Cursorless workflows

**For the Ecosystem:**
- Standardized interface for AI code manipulation
- Foundation for more sophisticated coding assistants
- Bridge between natural language and semantic code understanding

This approach transforms the challenge of LLM code targeting from a limitation into an opportunity—enabling more sophisticated, semantically-aware code manipulation than current text-based approaches while eliminating dependency on visual affordances that LLMs cannot access.

---

*This PRD provides the foundation for building a sophisticated MCP that leverages Cursorless's proven syntax tree manipulation capabilities. The remaining sections will detail the specific MCP implementation strategy.*
