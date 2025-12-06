# A22 Specification v0.1 (Foundational Draft)

**Status:** Draft
**License:** Apache 2.0
**Audience:** Runtime implementers, language tooling developers, contributors
**Goal:** Define the syntax, semantics, structures, and expectations of the A22 declarative agent language.

---

## 0. Design Principles

A22 is built around the following principles:

1.  **Declarative First**
    A22 describes *what* the agentic system is, not *how* it executes.
2.  **Composable Blocks**
    Everything is defined through reusable blocks: agents, tools, data, events, workflows.
3.  **Capability-Oriented**
    Agents declare capabilities, and runtimes supply implementations.
4.  **Stable, Minimal, Predictable**
    Small surface area, predictable evolution, conservative changes.
5.  **Portable and Vendor-Neutral**
    Any runtime should be able to execute a compliant A22 program.

---

## 1. File Structure

### 1.1 A22 Files
*   The default extension is `.a22`.
*   A22 files contain a sequence of declarations, each defining one block.

### 1.2 Valid Blocks

A22 defines the following blocks:
*   `agent` - Autonomous agent definition
*   `capability` - Abstract capability contract
*   `tool` - Callable function or API
*   `event` - Event definition
*   `workflow` - Step orchestration
*   `data` - Data schema definition
*   `config` - Runtime configuration and metadata
*   `provider` - Model provider configuration
*   `policy` - Security policy definition
*   `template` - Reusable prompt template

Blocks can appear in any order.

---

## 2. Syntax Overview

A22 uses a Terraform-style declarative syntax with:
*   Double-quoted identifiers for named resources
*   Curly-brace block structure
*   HCL-like expressions (strings, booleans, numbers, arrays, objects)

### 2.1 Example

```a22
agent "support_bot" {
    capabilities = ["memory", "retrieval"]

    on event "user_message" {
        use tool "embedder"
        call workflow "answer_user"
    }
}
```

---

## 3. Core Blocks

---

### 3.1 AGENT Block

Defines an autonomous or semi-autonomous agent with model configuration, security policy, and isolation settings.

**Syntax**

```a22
agent "<name>" {
    capabilities = [ ... ]
    state        = data.<schema>?

    # Simple model reference
    model = "<model-id>"?

    # OR: Advanced model configuration with fallback
    model {
        primary = {
            provider = provider.<name>
            name     = "<model-name>"
            params {
                temperature = <number>
                max_tokens  = <number>
                # ... provider-specific params
            }
        }
        fallback = [
            { provider = provider.<name>, name = "<model-name>" },
            # ... additional fallback providers
        ]
        strategy = "failover" | "cost_optimized" | "latency_optimized" | "capability_based"
    }

    # Security policy (optional)
    policy = policy.<name>?

    # OR: Inline policy
    policy {
        allow { ... }
        deny { ... }
        limits { ... }
    }

    # Isolation configuration (optional)
    isolation {
        memory     = "strict" | "shared" | "none"
        network    = "full" | "limited" | "none"
        filesystem = "full" | "readonly" | "none"
    }

    # System prompt (optional)
    system_prompt = "<prompt-text>"?

    # Event handlers
    on event "<event-name>" {
        call workflow "<workflow-name>"?
        use tool "<tool-name>"?
    }
}
```

**Fields**

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `capabilities` | `array(string)` | Yes | Declares required capabilities |
| `state` | `reference` | No | Data schema attached to agent memory |
| `model` | `string \| block` | No | Model configuration (simple string or advanced config) |
| `policy` | `reference \| block` | No | Security policy (reference or inline) |
| `isolation` | `block` | No | Isolation configuration for security boundaries |
| `system_prompt` | `string` | No | System-level prompt for the agent |
| `on event` | `block` | No | Event handler |

**Model Configuration**

Agents can specify models in two ways:

1. **Simple**: `model = "gpt-4"` - Runtime resolves to configured provider
2. **Advanced**: `model { ... }` - Explicit provider, fallback chain, and strategy

**Selection Strategies**:
*   `failover` (default) - Try primary, use fallback on error
*   `cost_optimized` - Choose cheapest provider meeting requirements
*   `latency_optimized` - Choose fastest provider based on historical data
*   `capability_based` - Route based on task complexity

**Isolation Levels**:
*   `strict` - Complete isolation, no shared resources
*   `shared` - Can access shared resources with permissions
*   `none` - No isolation (development only)

**Semantics**
*   Agents do nothing unless triggered by events or workflows
*   Capabilities must be supplied by runtime or bound at execution time
*   Model provider must be configured in runtime or declared via `provider` block
*   Security policies are enforced at runtime before any action
*   Isolation boundaries prevent unauthorized resource access

---

### 3.2 CAPABILITY Block

Defines an abstract capability an agent may require, with permission requirements and grants.

**Syntax**

```a22
capability "retrieval" {
    kind        = "external" | "system" | "builtin"
    description = "<description>"?

    # Required permissions (optional)
    requires {
        permissions = [
            { resource = "<resource>", action = "read" | "write" | "execute" | "admin" },
            # ... additional permissions
        ]
    }

    # What this capability grants access to (optional)
    grants {
        tools     = ["<tool-name>", ...]?
        workflows = ["<workflow-name>", ...]?
        data      = ["<data-type>", ...]?
    }
}
```

**Fields**

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `kind` | `string` | Yes | Capability type: external, system, or builtin |
| `description` | `string` | No | Human-readable description |
| `requires` | `block` | No | Permission requirements |
| `grants` | `block` | No | What resources this capability provides access to |

**Permission Actions**:
*   `read` - Read/retrieve data or resources
*   `write` - Create/modify data or resources
*   `execute` - Execute tools, workflows, or operations
*   `admin` - Full control over resources

**Resource Types**:
*   `tool.<name>` - Specific tool access
*   `workflow.<name>` - Specific workflow access
*   `data.<type>` - Data type access
*   `filesystem` - File system access
*   `network` - Network access
*   `memory` - Memory access
*   `*` - All resources (admin only)

**Semantics**
*   Capabilities are contracts only; implementations are runtime-specific
*   Runtime must check permission requirements before granting capability
*   Grants define what an agent can access when capability is bound

---

### 3.3 TOOL Block

Represents a callable function, API, or external executor with security constraints and sandboxing.

**Syntax**

```a22
tool "<name>" {
    schema {
        input  = data.<InputType>?
        output = data.<OutputType>?
        # OR: inline schema
        field1: string
        field2: number
        field3: array<string>
    }

    handler = external("<binding>")

    # Security configuration (optional)
    security {
        # Input validation
        validate {
            <field-name> = {
                max_length    = <number>?
                min_length    = <number>?
                pattern       = "<regex>"?
                deny_patterns = ["<pattern>", ...]?
                min           = <number>?  # For numeric fields
                max           = <number>?  # For numeric fields
            }
        }

        # Execution sandbox
        sandbox {
            timeout_ms        = <number>?
            max_memory_mb     = <number>?
            network_allowed   = <boolean>?
            network_hosts     = ["<host>", ...]?  # Whitelist
            filesystem_allowed = <boolean>?
            filesystem_paths  = ["<path>", ...]?  # Whitelist
            filesystem_mode   = "readonly" | "readwrite"?
        }

        # Output validation
        output {
            max_size_kb = <number>?
            schema      = data.<type>?  # Enforce schema match
        }
    }
}
```

**Fields**

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `schema` | `block` | Yes | Input/output schema definition |
| `handler` | `string` | Yes | External handler binding |
| `security` | `block` | No | Security configuration and sandboxing |

**Security Configuration**:

**`validate`**: Input validation rules per field
*   `max_length` / `min_length` - String length constraints
*   `pattern` - Regex pattern that must match
*   `deny_patterns` - Regex patterns that must NOT match (e.g., SQL injection)
*   `min` / `max` - Numeric range constraints

**`sandbox`**: Execution environment restrictions
*   `timeout_ms` - Maximum execution time in milliseconds
*   `max_memory_mb` - Maximum memory allocation
*   `network_allowed` - Whether network access is permitted
*   `network_hosts` - Whitelist of allowed hosts (requires `network_allowed = true`)
*   `filesystem_allowed` - Whether file system access is permitted
*   `filesystem_paths` - Whitelist of allowed paths (requires `filesystem_allowed = true`)
*   `filesystem_mode` - Read-only or read-write access

**`output`**: Output validation
*   `max_size_kb` - Maximum output size in kilobytes
*   `schema` - Data type schema that output must conform to

**Semantics**
*   Tools are side-effecting operations
*   Runtime determines actual function binding (cloud, local, API, etc)
*   Security validations are enforced before and after tool execution
*   Sandbox restrictions are enforced during tool execution
*   Violations result in tool execution failure

---

### 3.4 EVENT Block

Defines an event that can be emitted or listened for.

**Syntax**

```a22
event "user_message" {
    payload = data.UserMessage
}
```

---

### 3.5 WORKFLOW Block

Workflows orchestrate steps, tools, and agent calls.

**Syntax**

```a22
workflow "answer_user" {
    steps {
        embed  = tool "embedder" { text = input.text }
        search = capability "retrieval" { query = embed.vector }
        reply  = agent "support_bot" { context = search.docs }
    }
    returns = data.Answer
}
```

**Execution Semantics**
*   Steps run in order unless `parallel` keyword is used.
*   Steps reference tools, agents, or capabilities.
*   Workflows can emit events.

---

### 3.6 DATA Block

Defines structured data schemas.

**Syntax**

```a22
data Answer {
    text: string
    confidence: number
}
```

Supported primitive types:
*   `string`
*   `number`
*   `boolean`
*   `array<T>`
*   `object { ... }`

---

### 3.7 PROVIDER Block

Defines a model provider configuration with credentials, endpoints, and rate limits.

**Syntax**

```a22
provider "<name>" {
    type = "llm" | "embedding" | "vision" | "audio"

    # Credential reference (NOT actual keys)
    credentials = env.<VAR_NAME>
    # OR: Multiple credentials
    credentials {
        api_key = env.<VAR_NAME>
        org_id  = env.<VAR_NAME>
        # ... provider-specific credentials
    }

    # Provider-specific configuration
    config {
        endpoint = "<url>"?
        timeout  = <milliseconds>?
        retry    = <boolean>?
        # ... provider-specific config
    }

    # Rate limits (optional)
    limits {
        requests_per_minute = <number>?
        tokens_per_minute   = <number>?
        requests_per_day    = <number>?
    }
}
```

**Fields**

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `type` | `string` | Yes | Provider category: llm, embedding, vision, or audio |
| `credentials` | `reference \| block` | No | Environment variable reference(s) for credentials |
| `config` | `block` | No | Provider-specific configuration |
| `limits` | `block` | No | Rate limiting and quota configuration |

**Credential References**:

Credentials must ALWAYS reference environment variables or secrets, never literal strings:
*   `env.<VAR_NAME>` - Environment variable (e.g., `env.OPENAI_API_KEY`)
*   `secrets.<key_name>` - Secrets manager reference (runtime-specific)

**Parser Validation**: The parser MUST reject credential-like strings (e.g., starting with `sk-`, `api_`, etc.)

**Provider Types**:
*   `llm` - Large language models (text generation, chat)
*   `embedding` - Text embedding models
*   `vision` - Image/video understanding models
*   `audio` - Speech-to-text, text-to-speech models

**Semantics**
*   Providers define how to connect to model APIs
*   Runtime loads actual credentials from environment at execution time
*   Rate limits are enforced by runtime to prevent quota exhaustion
*   Provider configurations are reusable across multiple agents

**Example**:

```a22
provider "openai" {
    type = "llm"
    credentials = env.OPENAI_API_KEY

    config {
        endpoint = "https://api.openai.com/v1"
        timeout  = 30000
    }

    limits {
        requests_per_minute = 60
        tokens_per_minute   = 90000
    }
}
```

---

### 3.8 POLICY Block

Defines a security policy with access control rules and resource limits.

**Syntax**

```a22
policy "<name>" {
    # Whitelist of allowed resources
    allow {
        tools     = ["<tool-name>" | "*", ...]?
        workflows = ["<workflow-name>" | "*", ...]?
        data      = ["<data-type>" | "*", ...]?
        capabilities = ["<capability-name>", ...]?
    }

    # Blacklist of denied resources (overrides allow)
    deny {
        tools     = ["<tool-name>", ...]?
        workflows = ["<workflow-name>", ...]?
        data      = ["<data-type>", ...]?
    }

    # Resource limits
    limits {
        max_memory_mb      = <number>?
        max_execution_time = <milliseconds>?
        max_tool_calls     = <number>?
        max_workflow_depth = <number>?
    }
}
```

**Fields**

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `allow` | `block` | No | Whitelist of permitted resources |
| `deny` | `block` | No | Blacklist of forbidden resources (takes precedence) |
| `limits` | `block` | No | Resource consumption limits |

**Access Control**:

Policies use a **deny-by-default** model with explicit allow lists:
*   Resources not in `allow` are denied by default
*   Resources in `deny` are always forbidden (overrides `allow`)
*   Wildcards (`*`) grant access to all resources of that type
*   Specific resource names provide fine-grained control

**Resource Limits**:
*   `max_memory_mb` - Maximum memory allocation in megabytes
*   `max_execution_time` - Maximum total execution time in milliseconds
*   `max_tool_calls` - Maximum number of tool invocations
*   `max_workflow_depth` - Maximum workflow nesting depth

**Semantics**
*   Policies are enforced at runtime before any resource access
*   Deny rules always take precedence over allow rules
*   Limit violations cause immediate termination
*   Policies can be reused across multiple agents

**Example**:

```a22
policy "restricted" {
    allow {
        tools = ["web_search", "calculator"]
        workflows = ["simple_query"]
    }

    deny {
        tools = ["exec_shell", "file_delete"]
    }

    limits {
        max_memory_mb      = 512
        max_execution_time = 30000
        max_tool_calls     = 10
    }
}
```

---

### 3.9 TEMPLATE Block

Defines a reusable prompt template for agents.

**Syntax**

```a22
template "<name>" {
    system      = "<system-prompt>"?
    user_prefix = "<prefix>"?
    user_suffix = "<suffix>"?
    format      = "<format-string>"?
}
```

**Fields**

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `system` | `string` | No | System-level prompt text |
| `user_prefix` | `string` | No | Prefix added to user messages |
| `user_suffix` | `string` | No | Suffix added to user messages |
| `format` | `string` | No | Format string for message composition |

**Semantics**
*   Templates are reusable prompt configurations
*   Agents reference templates via `prompt_template = template.<name>`
*   Runtime applies template formatting to messages

**Example**:

```a22
template "research_assistant" {
    system = """
        You are a research assistant specializing in scientific literature.
        Always cite sources and maintain academic tone.
        Provide clear, evidence-based responses.
    """
    user_prefix = "Research Question: "
}
```

---

### 3.10 CONFIG Block Extensions

The `config` block supports runtime configuration for monitoring, auditing, and observability.

**Syntax**

```a22
# Audit logging configuration
config "audit" {
    enabled = <boolean>

    log_events = [
        "agent.start" | "agent.stop" | "agent.error" |
        "tool.call" | "tool.error" |
        "workflow.execute" | "workflow.error" |
        "permission.denied" | "credential.access",
        # ... additional events
    ]

    destination = "<uri>"  # file://, syslog://, cloudwatch://
    format      = "json" | "text" | "cef"
    retention_days = <number>?
    include_payloads = <boolean>?  # Security vs debugging tradeoff
}

# Monitoring configuration
config "monitoring" {
    usage_tracking = <boolean>?
    cost_tracking  = <boolean>?
    metrics        = <boolean>?
    tracing        = <boolean>?

    budget {
        daily_limit_usd  = <number>?
        alert_threshold  = <number>?  # 0.0 to 1.0
        hard_stop        = <boolean>?
    }

    health_check {
        interval_seconds = <number>?
        endpoint         = "<path>"?
    }

    alerts {
        latency_threshold_ms = <number>?
        error_rate_threshold = <number>?  # 0.0 to 1.0
    }
}
```

**Audit Events**:
*   `agent.start` / `agent.stop` / `agent.error` - Agent lifecycle
*   `tool.call` / `tool.error` - Tool execution
*   `workflow.execute` / `workflow.error` - Workflow execution
*   `permission.denied` - Security violations
*   `credential.access` - Credential usage

**Log Destinations**:
*   `file://./path/to/log` - Local file
*   `syslog://host:port` - Syslog server
*   `cloudwatch://log-group` - AWS CloudWatch (runtime-specific)

**Semantics**
*   Audit logs provide security and compliance trails
*   Monitoring tracks resource usage and costs
*   Budget limits prevent cost overruns
*   Health checks ensure system availability
*   Alerts notify operators of issues

---

## 4. Expressions & Values

A22 supports:
*   Strings: `"hello"`
*   Numbers: `42`, `3.14`
*   Booleans: `true`, `false`
*   Arrays: `[1, 2, 3]`
*   Objects:

```a22
{
    a = 1
    b = true
}
```

No custom functions or expressions allowed v1.0 — keeps language pure & deterministic.

---

## 5. Execution Model

### 5.1 Runtime Responsibilities

A runtime MUST:
1.  Parse & validate A22 files
2.  Resolve capabilities
3.  Bind tools to handlers
4.  Execute workflows in deterministic order
5.  Maintain agent state if defined
6.  Enforce sandboxing & security boundaries
7.  Support event dispatch & routing

### 5.2 Determinism
*   Workflow step ordering is deterministic.
*   Tools may be nondeterministic; A22 does not impose constraints.

### 5.3 Error Handling

Runtimes must:
*   Surface schema validation errors
*   Surface missing capability binding errors
*   Provide structured error messages

---

## 6. AST & IR Requirements

A22 languages must translate to a canonical IR containing:
*   Block types + names
*   Field bodies
*   Data schemas
*   Workflow step graph
*   Event routing table
*   Capability contract requirements

Runtimes must accept this IR structure.

AST structure is implementation-specific, but must be lossy of no data.

---

## 7. Validation Rules

A22 requires the following validation:

**Core Validation**:
1. Agents must reference valid events + workflows
2. Tools must reference valid schemas
3. Data schemas must be acyclic
4. Workflow steps must reference defined entities
5. Capabilities must match invocation shapes
6. No duplicate block names of same type within the same block type
7. No undeclared identifiers

**Model Gateway Validation**:
8. `provider` blocks must have unique names
9. `credentials` must reference `env.*` or `secrets.*`, never literal strings
10. Parser MUST reject credential-like strings (e.g., starting with `sk-`, `api_`, `key_`, etc.)
11. Provider references must resolve to declared providers
12. Model selection `strategy` must be one of: `failover`, `cost_optimized`, `latency_optimized`, `capability_based`, `round_robin`
13. If `model.fallback` is specified, `model.primary` must exist

**Security Validation**:
14. `policy` blocks must have unique names
15. Policy references must resolve to declared policies
16. `limits` values must be positive numbers
17. `isolation` levels must be one of: `strict`, `shared`, `none`
18. `sandbox.network_hosts` must be valid hostnames or IP addresses
19. `validate.pattern` and `validate.deny_patterns` must be valid regex
20. Permission `action` must be one of: `read`, `write`, `execute`, `admin`
21. Permission `resource` must be a valid resource identifier or wildcard
22. Denied resources in policy take precedence over allowed resources

**General Validation**:
23. `env.*` references must be valid identifier names (alphanumeric + underscore)
24. Audit log destinations must be valid URIs
25. Template references must resolve to declared templates
26. All block references must be resolvable at parse time or clearly marked as runtime-bound

---

## 8. Versioning

A22 follows semantic versioning:
*   `0.x` = not yet stable
*   `1.x` = stable language
*   `2.x` = potential breaking syntax changes

Backward compatibility is required unless otherwise stated.

---

## 9. Reserved Keywords

**Core Blocks**:
*   `agent`
*   `tool`
*   `capability`
*   `workflow`
*   `event`
*   `data`
*   `config`
*   `provider`
*   `policy`
*   `template`

**Workflow & Execution**:
*   `steps`
*   `parallel`
*   `returns`
*   `input`
*   `output`
*   `error_boundary`
*   `retry`
*   `on`

**Tool & Handler**:
*   `schema`
*   `handler`
*   `external`
*   `security`
*   `sandbox`
*   `validate`

**Model Gateway**:
*   `model`
*   `credentials`
*   `strategy`
*   `fallback`
*   `primary`
*   `params`
*   `env`
*   `secrets`
*   `limits`

**Security & Policy**:
*   `policy`
*   `allow`
*   `deny`
*   `isolation`
*   `requires`
*   `grants`
*   `permissions`

**Configuration**:
*   `audit`
*   `monitoring`
*   `metrics`
*   `tracing`
*   `alerts`
*   `budget`
*   `health_check`

**Data & Types**:
*   `string`
*   `number`
*   `boolean`
*   `array`
*   `object`

**State & Capability**:
*   `state`
*   `capabilities`
*   `kind`

**Template & Prompts**:
*   `system_prompt`
*   `prompt_template`
*   `system`
*   `user_prefix`
*   `user_suffix`
*   `format`

---

## 10. Security & Isolation

A22 provides a comprehensive security model with multiple layers of defense.

### 10.1 Security Principles

1. **Deny by Default**: Resources are inaccessible unless explicitly allowed
2. **Least Privilege**: Agents have minimal permissions needed for their function
3. **Defense in Depth**: Multiple security layers (policies, sandboxing, isolation)
4. **Audit Everything**: All security-relevant events are logged
5. **Credential Security**: API keys never appear in .a22 files

### 10.2 Security Layers

**Layer 1: Policy-Based Access Control**
*   Policies define allow/deny rules for tools, workflows, and data
*   Enforced before any resource access
*   Deny rules override allow rules
*   Resource limits prevent abuse

**Layer 2: Tool Sandboxing**
*   Input validation prevents injection attacks
*   Execution sandbox limits resources (CPU, memory, time)
*   Network and filesystem access restrictions
*   Output validation prevents data exfiltration

**Layer 3: Agent Isolation**
*   Memory isolation prevents cross-agent access
*   Network isolation controls external communication
*   Filesystem isolation restricts file access
*   State persistence controls prevent data leakage

**Layer 4: Audit Logging**
*   All security events logged
*   Tool invocations tracked
*   Permission denials recorded
*   Credential access monitored

### 10.3 Credential Management

**Critical Security Requirement**: Credentials MUST NEVER appear in .a22 files.

**Allowed**: Environment variable references
```a22
provider "openai" {
    credentials = env.OPENAI_API_KEY
}
```

**Forbidden**: Literal credential strings
```a22
provider "openai" {
    credentials = "sk-proj-abc123..."  # PARSER ERROR!
}
```

**Parser Validation**: Must reject strings matching credential patterns:
*   Starting with `sk-`, `key-`, `api_`, `secret_`
*   Pattern: `^(sk|key|api|secret)[-_]`
*   Any base64-encoded strings > 32 characters in credential fields

**Runtime Loading**:
1. Runtime reads .a22 file (contains only references)
2. Runtime loads environment variables
3. Runtime resolves `env.*` references at execution time
4. Credentials never logged or exposed

### 10.4 Runtime Enforcement

Runtimes MUST enforce:

**At Parse Time**:
*   Reject credential-like strings
*   Validate all policy and security configurations
*   Ensure all references are resolvable

**At Execution Time**:
*   Check policy permissions before resource access
*   Enforce sandbox restrictions during tool execution
*   Enforce isolation boundaries for agent memory
*   Track and limit resource consumption
*   Log all security-relevant events

**Execution Locality**:
*   A22 does not define where code runs (cloud, local, edge)
*   Security policies are portable across runtimes
*   Implementation-specific: container vs process vs VM isolation

### 10.5 Security Best Practices

**For Agent Builders**:
1. Always use policies to restrict agent capabilities
2. Set resource limits to prevent runaway execution
3. Use input validation for all tool parameters
4. Enable audit logging in production
5. Use strict isolation for untrusted agents
6. Deny dangerous tools by default (exec_shell, file_delete, etc.)

**For Runtime Implementers**:
1. Enforce credential validation at parse time
2. Implement proper sandbox isolation
3. Rate limit API calls to providers
4. Encrypt agent state at rest
5. Implement audit log rotation and retention
6. Provide security policy templates for common use cases

### 10.6 Threat Model

A22's security model protects against:

**Injection Attacks**:
*   SQL injection via input validation
*   Command injection via sandbox restrictions
*   Prompt injection via input sanitization

**Resource Abuse**:
*   DoS via execution time limits
*   Memory exhaustion via memory limits
*   API quota exhaustion via rate limiting

**Data Exfiltration**:
*   Output size limits
*   Network whitelisting
*   Filesystem access restrictions
*   Audit logging of data access

**Credential Theft**:
*   No credentials in .a22 files
*   Environment variable isolation
*   Audit logging of credential access

---

## 11. Complete Examples

### 11.1 Multi-Provider Agent with Fallback and Security

This example demonstrates:
- Multiple model providers with fallback
- Security policies with resource limits
- Tool sandboxing with input validation
- Audit logging

```a22
# Provider configurations
provider "openai" {
    type = "llm"
    credentials = env.OPENAI_API_KEY

    config {
        endpoint = "https://api.openai.com/v1"
        timeout  = 30000
    }

    limits {
        requests_per_minute = 60
        tokens_per_minute   = 90000
    }
}

provider "anthropic" {
    type = "llm"
    credentials = env.ANTHROPIC_API_KEY

    config {
        endpoint = "https://api.anthropic.com/v1"
    }
}

provider "local_ollama" {
    type = "llm"

    config {
        endpoint = "http://localhost:11434/api"
    }
}

# Security policy
policy "research_policy" {
    allow {
        tools = ["web_search", "document_retriever"]
        workflows = ["research_workflow"]
    }

    deny {
        tools = ["exec_shell", "file_delete"]
    }

    limits {
        max_memory_mb      = 1024
        max_execution_time = 120000  # 2 minutes
        max_tool_calls     = 20
    }
}

# Data schemas
data SearchQuery {
    query: string
    max_results: number
}

data SearchResult {
    title: string
    url: string
    snippet: string
}

data SearchResults {
    results: array<SearchResult>
}

data ResearchReport {
    summary: string
    sources: array<string>
    confidence: number
}

# Sandboxed tool
tool "web_search" {
    schema {
        input  = data.SearchQuery
        output = data.SearchResults
    }

    handler = external("https://api.search.example.com/search")

    security {
        validate {
            query = {
                max_length    = 500
                pattern       = "^[a-zA-Z0-9\\s\\-_]+$"
                deny_patterns = ["DROP TABLE", "SELECT *", "<script>"]
            }
            max_results = {
                min = 1
                max = 50
            }
        }

        sandbox {
            timeout_ms      = 10000
            max_memory_mb   = 128
            network_allowed = true
            network_hosts   = ["api.search.example.com"]
        }

        output {
            max_size_kb = 500
            schema      = data.SearchResults
        }
    }
}

# Agent with multi-provider fallback
agent "research_assistant" {
    capabilities = ["memory"]

    model {
        primary = {
            provider = provider.openai
            name     = "gpt-4-turbo"
            params {
                temperature = 0.7
                max_tokens  = 4096
            }
        }

        fallback = [
            {
                provider = provider.anthropic
                name     = "claude-3-5-sonnet"
            },
            {
                provider = provider.local_ollama
                name     = "llama-3.1-70b"
            }
        ]

        strategy = "failover"
    }

    policy = policy.research_policy

    isolation {
        memory     = "strict"
        network    = "limited"
        filesystem = "readonly"
    }

    system_prompt = """
        You are a research assistant specializing in academic research.
        Always cite sources, maintain objectivity, and provide evidence-based answers.
    """

    on event "research_request" {
        call workflow "research_workflow"
    }
}

# Event definition
event "research_request" {
    payload = data.SearchQuery
}

# Workflow
workflow "research_workflow" {
    steps {
        search = tool "web_search" {
            query       = input.query
            max_results = 10
        }

        analyze = agent "research_assistant" {
            context = search.results
        }
    }

    returns = data.ResearchReport
}

# Audit configuration
config "audit" {
    enabled = true

    log_events = [
        "agent.start",
        "agent.stop",
        "tool.call",
        "permission.denied"
    ]

    destination = "file://./logs/audit.jsonl"
    format      = "json"
    retention_days = 90
    include_payloads = false
}

# Monitoring configuration
config "monitoring" {
    usage_tracking = true
    cost_tracking  = true

    budget {
        daily_limit_usd = 50.0
        alert_threshold = 0.8
        hard_stop       = true
    }
}
```

### 11.2 Minimal Example

A minimal A22 program with basic agent and workflow:

```a22
data Question {
    text: string
}

data Answer {
    text: string
}

provider "openai" {
    type = "llm"
    credentials = env.OPENAI_API_KEY
}

event "user_question" {
    payload = data.Question
}

tool "generate_answer" {
    schema {
        prompt: string
    }
    handler = external("model.generate")
}

workflow "qa" {
    steps {
        answer = tool "generate_answer" { prompt = input.text }
    }
    returns = data.Answer
}

agent "qa_bot" {
    model = provider.openai.gpt-4

    on event "user_question" {
        call workflow "qa"
    }
}
```

### 11.3 Secure Multi-Agent System

Example of multiple agents with different security policies:

```a22
# Admin policy
policy "admin" {
    allow {
        tools = ["*"]
        workflows = ["*"]
    }

    limits {
        max_memory_mb = 2048
    }
}

# Analyst policy
policy "analyst" {
    allow {
        tools = ["data_query", "chart_generator"]
        workflows = ["analysis_workflow"]
    }

    deny {
        tools = ["data_delete", "exec_shell"]
    }

    limits {
        max_memory_mb      = 1024
        max_execution_time = 60000
        max_tool_calls     = 50
    }
}

# Public policy
policy "public" {
    allow {
        tools = ["web_search"]
        workflows = ["public_query"]
    }

    limits {
        max_memory_mb      = 256
        max_execution_time = 10000
        max_tool_calls     = 5
    }
}

# Admin agent
agent "admin_agent" {
    model = provider.openai.gpt-4
    policy = policy.admin

    isolation {
        memory = "shared"  # Can access shared resources
    }
}

# Analyst agent
agent "analyst_agent" {
    model = provider.openai.gpt-3.5-turbo
    policy = policy.analyst

    isolation {
        memory = "strict"  # Isolated memory
        network = "limited"
    }
}

# Public agent
agent "public_agent" {
    model = provider.local_ollama.llama-3.1-8b
    policy = policy.public

    isolation {
        memory     = "strict"
        network    = "limited"
        filesystem = "none"
    }
}
```

---

## 12. Credential Management Best Practices

### 12.1 Environment Variables

Store credentials in `.env` files (never committed to git):

```bash
# .env
OPENAI_API_KEY=sk-proj-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_AI_KEY=...
```

Reference in A22:

```a22
provider "openai" {
    type = "llm"
    credentials = env.OPENAI_API_KEY
}
```

### 12.2 Secrets Managers

For production, use secrets managers:

```a22
provider "openai" {
    type = "llm"
    credentials = secrets.openai_key  # Runtime resolves from secrets manager
}
```

### 12.3 Development vs Production

**Development**: Use `.env` files locally
```bash
# Load .env
source .env

# Run A22
a22 run app.a22
```

**Production**: Use environment variables or secrets manager
```bash
# Set env vars in container/VM
export OPENAI_API_KEY=...

# Or use secrets manager (AWS, GCP, Azure)
a22 run app.a22 --secrets aws-secrets-manager
```

---

## 13. Summary

A22 provides a comprehensive, declarative language for building secure, production-ready agentic systems.

**Key Features**:
*   **Universal Model Gateway**: Multi-provider support with fallback and cost optimization
*   **Security First**: Policy-based access control, sandboxing, isolation, audit logging
*   **Builder-Friendly**: Simple syntax, visual mental model, reusable blocks
*   **Vendor-Neutral**: No lock-in to specific providers or platforms
*   **Production-Ready**: Resource limits, monitoring, cost management, compliance

**For Builders**: Define agent behavior declaratively without writing code
**For Runtime Implementers**: Clear specification for parsing, validation, and execution
**For Security Teams**: Comprehensive security model with defense in depth

**Next Steps**:
1. Review the specification
2. Implement parsers and runtimes
3. Build secure, scalable agentic systems
