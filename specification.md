# A22 Language Specification v0.1

## 1. Introduction
A22 is a declarative, agent-native language for defining agentic systems. It separates the **definition** of agent behavior from its **execution**, ensuring portability, determinism, and a clear mental model for builders.

## 2. Core Concepts

### 2.1 Capability vs Resource Model
A22 distinguishes between **Capabilities** (what an agent *can do*) and **Resources** (what an agent *has access to*).

*   **Capabilities (Tools)**: Immutable, stateless functions or skills.
    *   *Example*: `web_search`, `calculator`, `send_email`.
    *   Defined once, reused across agents.
*   **Resources (Memory/State)**: Mutable, stateful entities.
    *   *Example*: `conversation_history`, `user_profile_db`, `file_system`.
    *   Agents read from and write to resources.
*   **Agents**: The actors that bind capabilities and resources to a model and a directive (system prompt).

### 2.2 Execution Graph (ADG)
An A22 system compiles down to an **Acyclic Directed Graph (ADG)**.
*   **Nodes**: Agents, Routers, Tools.
*   **Edges**: Event flows (Message passing).
*   **Execution**:
    1.  **Trigger**: An event enters the graph.
    2.  **Flow**: Data flows through routes to agents.
    3.  **Step**: An agent executes (Model + Tools + Memory).
    4.  **Output**: Result flows to the next node or final output.

## 3. Syntax & Grammar

A22 uses a HCL-like syntax optimized for readability.

```ebnf
config      = { block }
block       = type identifier [ label ] "{" body "}"
body        = { attribute | block }
attribute   = key "=" expression
expression  = literal | reference | list | map | call
```

### 3.1 Blocks
Top-level structures that define system components.
*   `agent`: Defines an actor.
*   `tool`: Defines a capability.
*   `resource`: Defines a stateful backend (Memory).
*   `input` / `output`: Defines the interface.
*   `router`: Defines control flow logic.

### 3.2 Types
*   `string`: "hello"
*   `number`: 42, 3.14
*   `bool`: true, false
*   `list`: ["a", "b"]
*   `map`: { key = "val" }
*   `ref`: agent.name, tool.search

## 4. Block Definitions

### 4.1 Tool (Capability)
Defines a functional capability.

```a22
tool "web_search" {
  description = "Search the internet for up-to-date information"
  
  # Implementation reference (URI or path)
  source = "std/web_search"

  inputs {
    query = string
  }
  outputs {
    results = list(string)
  }
}
```

### 4.2 Resource (Memory)
Defines a storage backend.

```a22
resource "chat_history" {
  type = "vector_store"
  provider = "pinecone" # or "local", "postgres"
  ttl = "24h"
}
```

### 4.3 Agent
Binds Model, Tools, and Resources.

```a22
agent "researcher" {
  model = "gpt-4-turbo"
  
  system_prompt = <<EOF
    You are a researcher. Use the web_search tool to find information.
    Always cite your sources.
  EOF

  # Capabilities
  use "web_search" {
    tool = tool.web_search
  }

  # Resources
  memory "short_term" {
    resource = resource.chat_history
    mode = "read_write"
  }
}
```

### 4.4 Router (Control Flow)
Directs traffic based on logic.

```a22
router "triage" {
  entry = true # Entry point of the graph

  route {
    if = contains(input.text, "help")
    to = agent.support
  }

  route {
    default = true
    to = agent.researcher
  }
}
```

## 5. Semantics

### 5.1 Scoping & Visibility
*   Global Scope: All top-level blocks are visible to each other.
*   Agent Scope: Tools and Resources defined/linked inside an agent are only accessible to that agent.

### 5.2 Determinism
*   **Static Graph**: The structure of the graph (nodes and edges) is constant at compile time.
*   **Dynamic Execution**: The path taken *through* the graph depends on runtime data, but the *possible* paths are fixed.

## 6. AST Structure

The Abstract Syntax Tree represents the parsed configuration.

```typescript
// Root
interface Program {
  kind: "Program";
  blocks: Block[];
}

// Generic Block
interface Block {
  kind: "Block";
  type: string;       // "agent", "tool", "resource"
  identifier: string; // "researcher"
  label?: string;     // Optional secondary label
  attributes: Attribute[];
  children: Block[];  // Nested blocks
}

// Attribute
interface Attribute {
  kind: "Attribute";
  key: string;
  value: Expression;
}

// Expressions
type Expression = 
  | Literal 
  | Reference 
  | ListExpr 
  | MapExpr;

interface Reference {
  kind: "Reference";
  path: string[]; // ["tool", "web_search"]
}
```

## 7. JSON IR (Intermediate Representation)
The compiler transpiles A22 code into a portable JSON format for runtimes.

```json
{
  "version": "0.1",
  "graph": {
    "nodes": {
      "agent.researcher": {
        "type": "agent",
        "model": "gpt-4-turbo",
        "tools": ["tool.web_search"]
      },
      "tool.web_search": {
        "type": "tool",
        "source": "std/web_search"
      }
    },
    "edges": [
      { "from": "router.triage", "to": "agent.researcher" }
    ]
  }
}
```
