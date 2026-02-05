# Design Document: CodeChronicle

## Overview

CodeChronicle is a VS Code extension that combines deterministic static analysis with AI-powered semantic reasoning to help developers understand and safely modify legacy codebases. The architecture follows a clear separation of concerns: deterministic logic handles file scanning, dependency detection, and graph construction, while AWS AI services provide semantic understanding, risk assessment, and natural language interaction.

The extension consists of three main layers:
1. **Analysis Layer**: Scans the workspace, parses dependencies, and constructs the code graph
2. **AI Layer**: Interfaces with AWS Bedrock to provide summaries, risk analysis, and natural language queries
3. **Visualization Layer**: Renders an interactive graph in a VS Code webview

## Architecture

### High-Level Architecture

```mermaid
graph TB
    subgraph "VS Code Extension Host"
        ExtensionMain[Extension Main]
        FileWatcher[File System Watcher]
        CommandHandler[Command Handler]
    end
    
    subgraph "Analysis Layer"
        Scanner[Workspace Scanner]
        Parser[Dependency Parser]
        GraphBuilder[Graph Builder]
        MetricsEngine[Metrics Engine]
        Cache[Cache Manager]
    end
    
    subgraph "AI Layer"
        AIClient[AWS Bedrock Client]
        SummaryService[Summary Service]
        RiskService[Risk Assessment Service]
        QueryService[Query Service]
    end
    
    subgraph "Visualization Layer"
        WebviewProvider[Webview Provider]
        GraphRenderer[Graph Renderer]
        UIController[UI Controller]
    end
    
    ExtensionMain --> Scanner
    ExtensionMain --> CommandHandler
    FileWatcher --> GraphBuilder
    Scanner --> Parser
    Parser --> GraphBuilder
    GraphBuilder --> MetricsEngine
    GraphBuilder --> Cache
    
    CommandHandler --> WebviewProvider
    WebviewProvider --> GraphRenderer
    GraphRenderer --> UIController
    
    UIController --> SummaryService
    UIController --> RiskService
    UIController --> QueryService
    
    SummaryService --> AIClient
    RiskService --> AIClient
    QueryService --> AIClient
    
    AIClient --> AWS[AWS Bedrock]
```

### Component Responsibilities

**Extension Main**
- Activates the extension when VS Code starts or when a relevant command is invoked
- Registers commands, file watchers, and webview providers
- Manages extension lifecycle and cleanup

**Workspace Scanner**
- Discovers all source files in the workspace
- Respects .gitignore patterns and configured exclusions
- Returns a list of file paths for analysis

**Dependency Parser**
- Extracts import/require statements from source files using regex
- Supports multiple languages (JavaScript, TypeScript, Python, Java, etc.)
- Resolves relative imports to absolute workspace paths

**Graph Builder**
- Constructs a directed graph from parsed dependencies
- Maintains nodes (files) and edges (dependencies)
- Provides graph traversal methods for blast radius computation

**Metrics Engine**
- Computes structural metrics: dependency count, dependent count, lines of code
- Calculates centrality scores using graph algorithms
- Provides deterministic complexity measures

**Cache Manager**
- Serializes and deserializes the code graph to disk
- Caches AI-generated summaries with content hashes
- Invalidates cache entries when files change

**AWS Bedrock Client**
- Manages authentication and connection to AWS Bedrock
- Handles rate limiting and retry logic
- Provides a unified interface for AI service calls

**Summary Service**
- Generates human-readable file summaries using AI
- Includes context about file dependencies and role in the codebase
- Caches results to minimize API calls

**Risk Assessment Service**
- Analyzes files for semantic risk factors
- Considers business logic, side effects, security sensitivity
- Returns risk scores (low, medium, high) with explanations

**Query Service**
- Processes natural language queries about the codebase
- Uses graph context to provide accurate answers
- Returns file-level and line-level references

**Webview Provider**
- Creates and manages the VS Code webview panel
- Handles communication between extension and webview
- Passes graph data and user interactions

**Graph Renderer**
- Renders the code graph using a JavaScript graph library (e.g., Cytoscape.js)
- Applies visual encodings: size for complexity, color for risk
- Handles user interactions: clicks, hovers, zoom, pan

**UI Controller**
- Manages webview state and user interactions
- Coordinates between graph visualization and side panels
- Handles blast radius mode and query interface

## Components and Interfaces

### Core Data Structures

#### CodeGraph
```typescript
interface CodeGraph {
  nodes: Map<string, GraphNode>;
  edges: GraphEdge[];
  metadata: GraphMetadata;
}

interface GraphNode {
  id: string;                    // Unique file identifier
  path: string;                  // Workspace-relative path
  metrics: StructuralMetrics;
  riskFactor?: RiskFactor;
  summary?: string;
  contentHash: string;           // For cache invalidation
}

interface GraphEdge {
  source: string;                // Source node ID
  target: string;                // Target node ID
  type: DependencyType;          // import, require, include
}

interface GraphMetadata {
  workspacePath: string;
  totalFiles: number;
  lastUpdated: Date;
  version: string;
}
```

#### StructuralMetrics
```typescript
interface StructuralMetrics {
  linesOfCode: number;
  dependencyCount: number;       // Outgoing edges
  dependentCount: number;        // Incoming edges
  centralityScore: number;       // Betweenness centrality
}
```

#### RiskFactor
```typescript
interface RiskFactor {
  level: 'low' | 'medium' | 'high';
  score: number;                 // 0-100
  explanation: string;
  factors: string[];             // e.g., ["database access", "authentication logic"]
}
```

### Analysis Layer Interfaces

#### IWorkspaceScanner
```typescript
interface IWorkspaceScanner {
  scan(workspacePath: string, exclusions: string[]): Promise<string[]>;
}
```

#### IDependencyParser
```typescript
interface IDependencyParser {
  parse(filePath: string, content: string): Dependency[];
  getSupportedExtensions(): string[];
}

interface Dependency {
  importPath: string;
  resolvedPath?: string;
  lineNumber: number;
}
```

#### IGraphBuilder
```typescript
interface IGraphBuilder {
  buildGraph(files: string[], dependencies: Map<string, Dependency[]>): CodeGraph;
  updateNode(nodeId: string, content: string): void;
  removeNode(nodeId: string): void;
  computeBlastRadius(nodeId: string): string[];
}
```

#### IMetricsEngine
```typescript
interface IMetricsEngine {
  computeMetrics(graph: CodeGraph): void;
  computeNodeMetrics(node: GraphNode, graph: CodeGraph): StructuralMetrics;
}
```

### AI Layer Interfaces

#### IAWSBedrockClient
```typescript
interface IAWSBedrockClient {
  initialize(credentials: AWSCredentials): Promise<void>;
  invokeModel(modelId: string, prompt: string): Promise<string>;
  checkRateLimit(): boolean;
}

interface AWSCredentials {
  accessKeyId: string;
  secretAccessKey: string;
  region: string;
}
```

#### ISummaryService
```typescript
interface ISummaryService {
  generateSummary(node: GraphNode, graph: CodeGraph): Promise<string>;
  getCachedSummary(nodeId: string, contentHash: string): string | null;
}
```

#### IRiskAssessmentService
```typescript
interface IRiskAssessmentService {
  assessRisk(node: GraphNode, graph: CodeGraph): Promise<RiskFactor>;
}
```

#### IQueryService
```typescript
interface IQueryService {
  processQuery(query: string, graph: CodeGraph): Promise<QueryResult>;
}

interface QueryResult {
  answer: string;
  references: FileReference[];
  suggestedQuestions?: string[];
}

interface FileReference {
  nodeId: string;
  path: string;
  lineNumbers?: number[];
  snippet?: string;
}
```

### Visualization Layer Interfaces

#### IWebviewProvider
```typescript
interface IWebviewProvider {
  createWebview(graph: CodeGraph): void;
  updateGraph(graph: CodeGraph): void;
  handleMessage(message: WebviewMessage): void;
}

interface WebviewMessage {
  command: string;
  payload: any;
}
```

#### Message Protocol (Extension ↔ Webview)
```typescript
// Extension → Webview
type ExtensionMessage =
  | { type: 'init', graph: CodeGraph }
  | { type: 'update', graph: CodeGraph }
  | { type: 'highlight', nodeIds: string[] }
  | { type: 'summary', nodeId: string, summary: string }
  | { type: 'risk', nodeId: string, risk: RiskFactor }
  | { type: 'queryResult', result: QueryResult };

// Webview → Extension
type WebviewMessage =
  | { type: 'nodeClick', nodeId: string }
  | { type: 'blastRadius', nodeId: string }
  | { type: 'query', query: string }
  | { type: 'openFile', path: string }
  | { type: 'refresh' };
```

## Data Models

### File System Representation

The extension works directly with the VS Code workspace file system. Files are identified by their workspace-relative paths, which serve as unique identifiers in the graph.

### Graph Storage

The code graph is stored in two formats:

1. **In-Memory**: A `CodeGraph` object maintained during the extension session
2. **Persistent Cache**: A JSON file stored in `.vscode/code-chronicle/graph-cache.json`

Cache structure:
```json
{
  "version": "1.0.0",
  "workspacePath": "/path/to/workspace",
  "lastUpdated": "2024-01-15T10:30:00Z",
  "nodes": {
    "src/index.ts": {
      "id": "src/index.ts",
      "path": "src/index.ts",
      "contentHash": "abc123...",
      "metrics": {
        "linesOfCode": 150,
        "dependencyCount": 5,
        "dependentCount": 2,
        "centralityScore": 0.75
      },
      "riskFactor": {
        "level": "medium",
        "score": 65,
        "explanation": "Entry point with database access",
        "factors": ["database access", "authentication"]
      },
      "summary": "Main application entry point..."
    }
  },
  "edges": [
    {
      "source": "src/index.ts",
      "target": "src/database.ts",
      "type": "import"
    }
  ]
}
```

### AI Service Prompts

#### Summary Generation Prompt Template
```
You are analyzing a source code file in a large codebase.

File: {filePath}
Lines of Code: {linesOfCode}
Dependencies: {dependencyCount}
Dependents: {dependentCount}

File Content:
{fileContent}

Dependency Context:
This file imports: {importedFiles}
This file is imported by: {dependentFiles}

Generate a concise summary (2-3 sentences) explaining:
1. What this file does
2. Why it exists in the codebase
3. Its role in the overall architecture
```

#### Risk Assessment Prompt Template
```
You are assessing the risk of modifying a source code file.

File: {filePath}
Structural Metrics:
- Lines of Code: {linesOfCode}
- Number of Dependencies: {dependencyCount}
- Number of Dependents: {dependentCount}
- Centrality Score: {centralityScore}

File Content:
{fileContent}

Analyze this file for semantic risk factors including:
- Business-critical logic
- Side effects (database writes, API calls, file I/O)
- Security-sensitive operations (authentication, authorization, encryption)
- Hidden coupling (global state, singletons, event emitters)

Return a JSON object with:
{
  "level": "low" | "medium" | "high",
  "score": 0-100,
  "explanation": "Brief explanation of the risk",
  "factors": ["factor1", "factor2", ...]
}
```

#### Query Processing Prompt Template
```
You are answering questions about a codebase.

User Query: {query}

Codebase Context:
Total Files: {totalFiles}
Graph Structure: {graphSummary}

Relevant Files:
{relevantFiles}

Answer the user's question with:
1. A clear, concise answer
2. Specific file references with paths
3. Line numbers if applicable
4. Suggested follow-up questions if helpful

Format your response as JSON:
{
  "answer": "Your answer here",
  "references": [
    {
      "path": "src/file.ts",
      "lineNumbers": [10, 15],
      "snippet": "relevant code snippet"
    }
  ],
  "suggestedQuestions": ["question1", "question2"]
}
```

### Dependency Parsing Patterns

The extension uses language-specific regex patterns to extract dependencies:

**JavaScript/TypeScript**
```typescript
const patterns = [
  /import\s+.*\s+from\s+['"](.+)['"]/g,           // ES6 imports
  /import\s+['"](.+)['"]/g,                       // Side-effect imports
  /require\s*\(\s*['"](.+)['"]\s*\)/g,           // CommonJS requires
  /import\s*\(\s*['"](.+)['"]\s*\)/g             // Dynamic imports
];
```

**Python**
```typescript
const patterns = [
  /^import\s+(\S+)/gm,                            // import module
  /^from\s+(\S+)\s+import/gm                      // from module import
];
```

**Java**
```typescript
const patterns = [
  /^import\s+([\w.]+);/gm                         // import statements
];
```

The parser resolves relative imports to absolute workspace paths and filters out external packages (node_modules, site-packages, etc.).

## Error Handling

### Error Categories

1. **File System Errors**: File not found, permission denied, invalid path
2. **Parsing Errors**: Malformed code, unsupported syntax, encoding issues
3. **AI Service Errors**: Rate limiting, network failures, invalid responses
4. **Graph Errors**: Circular dependencies, orphaned nodes, invalid references

### Error Handling Strategy

**File System Errors**
- Log the error with file path and reason
- Continue processing remaining files
- Display a notification to the user if critical files fail

**Parsing Errors**
- Log the error with file path and line number
- Skip the problematic file and continue
- Mark the node as "unparsed" in the graph

**AI Service Errors**
- Implement exponential backoff for rate limiting
- Cache and return previous results if available
- Fall back to structural metrics only
- Display user-friendly error messages in the UI

**Graph Errors**
- Validate graph structure after construction
- Remove orphaned nodes and invalid edges
- Log warnings for circular dependencies

### Graceful Degradation

The extension is designed to degrade gracefully when AI services are unavailable:

1. **No AWS Credentials**: Disable AI features, show only structural metrics
2. **Rate Limit Exceeded**: Queue requests, show cached results, notify user
3. **Network Failure**: Continue with deterministic features (graph, metrics)
4. **Invalid AI Response**: Log error, show structural data, retry on next request

## Testing Strategy

### Unit Testing

Unit tests will verify individual components in isolation:

- **Workspace Scanner**: Test file discovery with various .gitignore patterns
- **Dependency Parser**: Test regex patterns against sample code snippets
- **Graph Builder**: Test graph construction with known dependency structures
- **Metrics Engine**: Test metric calculations with predefined graphs
- **Cache Manager**: Test serialization, deserialization, and invalidation

### Integration Testing

Integration tests will verify component interactions:

- **End-to-End Graph Construction**: Scan → Parse → Build → Compute Metrics
- **AI Service Integration**: Mock AWS Bedrock responses and verify handling
- **Webview Communication**: Test message passing between extension and webview
- **File Watcher Integration**: Test incremental updates when files change

### Property-Based Testing

Property-based tests will verify universal correctness properties across many generated inputs. Each property test will run a minimum of 100 iterations with randomized inputs.


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property 1: File Discovery Completeness
*For any* workspace with a set of source files and .gitignore patterns, scanning should discover all files that are not excluded by .gitignore patterns or common dependency directories (node_modules, vendor, .git), and all discovered file paths should be relative to the workspace root.

**Validates: Requirements 1.1, 1.2, 1.4**

### Property 2: Import Extraction Accuracy
*For any* source file containing valid import statements, parsing should extract all import statements using language-specific patterns, and each extracted import should preserve its original import path and line number.

**Validates: Requirements 2.1**

### Property 3: Local vs External Import Classification
*For any* import statement, the parser should correctly classify it as either local (creating an edge in the graph) or external (excluded from the graph), and no external package imports should appear as edges in the code graph.

**Validates: Requirements 2.2, 2.3**

### Property 4: Graph Structure Consistency
*For any* set of parsed files and dependencies, the constructed code graph should have exactly one node per file, and each dependency relationship should correspond to exactly one edge from the importing file to the imported file.

**Validates: Requirements 2.4**

### Property 5: Error Resilience in Parsing
*For any* set of files where some files have parsing errors, the extension should log errors for unparseable files and successfully process all remaining parseable files, resulting in a partial graph that includes all successfully parsed files.

**Validates: Requirements 2.5, 11.3**

### Property 6: Comprehensive Metrics Computation
*For any* node in the code graph, the extension should compute and store all structural metrics (incoming edge count, outgoing edge count, lines of code, centrality score) as node metadata, and these metrics should be accessible for all nodes.

**Validates: Requirements 3.1, 3.2, 3.3, 3.4, 3.5**

### Property 7: AI Request Context Completeness
*For any* AI service request (summary, risk assessment, or query), the request should include all necessary context: file contents, structural metrics, and relevant graph information (dependencies and dependents).

**Validates: Requirements 4.2, 6.2, 8.1**

### Property 8: Risk Factor Extraction
*For any* valid AI service response containing risk analysis, the extension should successfully extract both the risk level (low, medium, high) and a human-readable explanation.

**Validates: Requirements 4.3, 4.4**

### Property 9: Visual Encoding Consistency
*For any* node rendered in the webview, the node size should be proportional to its complexity score, and the node color should correspond to its risk level (green for low, yellow for medium, red for high).

**Validates: Requirements 5.3, 5.4**

### Property 10: Blast Radius Completeness
*For any* node in the code graph, computing the blast radius should return all nodes that transitively depend on the selected node, including both direct and indirect dependents.

**Validates: Requirements 7.1, 7.2**

### Property 11: Blast Radius Round Trip
*For any* visualization state, activating blast radius mode and then deactivating it should restore the original visualization state with no nodes remaining highlighted.

**Validates: Requirements 7.5**

### Property 12: Query Response Structure
*For any* natural language query that AI service can answer, the response should include a textual answer and file-level references, and all referenced files should be highlighted in the webview.

**Validates: Requirements 8.2, 8.3**

### Property 13: Graph Serialization Round Trip
*For any* code graph, serializing it to a cache file and then deserializing it should produce an equivalent graph with the same nodes, edges, and metadata.

**Validates: Requirements 9.1, 9.3**

### Property 14: Cache Invalidation on Content Change
*For any* file with cached AI analysis, modifying the file content should invalidate the cached analysis, and subsequent requests should trigger new AI service calls rather than returning stale cached data.

**Validates: Requirements 9.4, 10.4**

### Property 15: Summary Caching Effectiveness
*For any* file that has been summarized, requesting the summary again without modifying the file should return the cached result without making a new AI service call.

**Validates: Requirements 6.4, 9.2**

### Property 16: Incremental Node Addition
*For any* code graph and any new file created in the workspace, the extension should add a new node to the graph with correct dependencies, and the webview should reflect the updated graph.

**Validates: Requirements 10.1, 10.5**

### Property 17: Incremental Node Removal
*For any* code graph and any file deleted from the workspace, the extension should remove the corresponding node and all edges connected to that node (both incoming and outgoing), and the webview should reflect the updated graph.

**Validates: Requirements 10.2, 10.5**

### Property 18: Incremental Dependency Update
*For any* file in the graph, modifying its import statements should result in the graph edges being updated to reflect the new dependencies, with old edges removed and new edges added as appropriate.

**Validates: Requirements 10.3**

### Property 19: AI Service Fallback Behavior
*For any* AI service call that fails, the extension should return cached results if available, or fall back to displaying structural metrics only, and should never crash or leave the UI in an inconsistent state.

**Validates: Requirements 4.5, 11.2**

### Property 20: Status Bar State Accuracy
*For any* extension state (scanning, ready, error), the status bar item should display the current state, and state transitions should be reflected in the status bar within 1 second.

**Validates: Requirements 12.4**

### Property 21: Configuration Exclusion Patterns
*For any* configured file exclusion patterns, scanning the workspace should exclude all files matching those patterns, and no excluded files should appear as nodes in the code graph.

**Validates: Requirements 13.1**

### Property 22: Configuration File Limit
*For any* configured maximum file limit, the extension should analyze at most that many files, and should not exceed the limit even if more files exist in the workspace.

**Validates: Requirements 13.4**
