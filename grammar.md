# A22 Grammar (EBNF) — v1.0

## 1. Top-Level Structure

```ebnf
a22_file     ::= declaration*
declaration  ::= agent_decl
               | tool_decl
               | capability_decl
               | workflow_decl
               | event_decl
               | data_decl
               | config_decl
               | provider_decl
               | policy_decl
               | template_decl
```

---

## 2. Lexical Elements

### 2.1 Identifiers

```ebnf
IDENT        ::= QUOTED_STRING
QUOTED_STRING ::= '"' (CHAR_NO_QUOTE)* '"'
CHAR_NO_QUOTE ::= /* any Unicode char except " */
```

A22 requires double-quoted identifiers for clarity.

### 2.2 Literals

```ebnf
string_lit   ::= QUOTED_STRING
number_lit   ::= DIGITS ('.' DIGITS)?
boolean_lit  ::= 'true' | 'false'

DIGITS       ::= [0-9]+
```

### 2.3 Values

```ebnf
value        ::= string_lit
               | number_lit
               | boolean_lit
               | array_lit
               | object_lit
               | reference

array_lit    ::= '[' (value (',' value)*)? ']'

object_lit   ::= '{' (object_field)* '}'
object_field ::= IDENT '=' value
```

### 2.4 References

```ebnf
reference    ::= IDENT ('.' IDENT)*
```

Examples:
*   `data.UserMessage`
*   `tool.embedder`

---

## 3. Blocks

### 3.1 Agent Block

```ebnf
agent_decl ::= 'agent' IDENT '{' agent_body '}'

agent_body ::= ( agent_field
               | event_handler_block
               | model_block
               | policy_block
               | isolation_block
               )*

agent_field ::= 'capabilities' '=' array_lit
               | 'state' '=' reference
               | 'model' '=' (string_lit | reference)
               | 'policy' '=' reference
               | 'system_prompt' '=' string_lit
               | 'prompt_template' '=' reference

# Advanced model configuration
model_block ::= 'model' '{' model_config '}'

model_config ::= ( 'primary' '=' model_provider_config
                 | 'fallback' '=' array_of_provider_configs
                 | 'strategy' '=' strategy_value
                 )*

model_provider_config ::= '{'
                            'provider' '=' reference ','
                            'name' '=' string_lit ','?
                            ('params' '{' param_field* '}')?
                          '}'

array_of_provider_configs ::= '[' (model_provider_config (',' model_provider_config)*)? ']'

strategy_value ::= 'failover' | 'cost_optimized' | 'latency_optimized'
                 | 'capability_based' | 'round_robin'

param_field ::= IDENT '=' value

# Inline policy block
policy_block ::= 'policy' '{' policy_body '}'

# Isolation block
isolation_block ::= 'isolation' '{' isolation_field* '}'

isolation_field ::= 'memory' '=' isolation_level
                  | 'network' '=' isolation_level
                  | 'filesystem' '=' filesystem_level

isolation_level ::= 'strict' | 'shared' | 'none'
filesystem_level ::= 'full' | 'readonly' | 'none'

# Event handlers
event_handler_block ::= 'on' 'event' IDENT '{' event_handler_body '}'

event_handler_body ::= (workflow_call | tool_call)*

workflow_call ::= 'call' 'workflow' IDENT
tool_call      ::= 'use'  'tool'      IDENT
```

### 3.2 Capability Block

```ebnf
capability_decl ::= 'capability' IDENT '{' capability_body '}'

capability_body ::= capability_field*

capability_field ::= 'kind' '=' capability_kind
                   | 'description' '=' string_lit
                   | requires_block
                   | grants_block

capability_kind ::= 'external' | 'system' | 'builtin'

requires_block ::= 'requires' '{' requires_field* '}'

requires_field ::= 'permissions' '=' array_of_permissions

array_of_permissions ::= '[' (permission_object (',' permission_object)*)? ']'

permission_object ::= '{'
                        'resource' '=' string_lit ','
                        'action' '=' permission_action
                      '}'

permission_action ::= 'read' | 'write' | 'execute' | 'admin'

grants_block ::= 'grants' '{' grants_field* '}'

grants_field ::= 'tools' '=' array_lit
               | 'workflows' '=' array_lit
               | 'data' '=' array_lit
```

### 3.3 Tool Block

```ebnf
tool_decl ::= 'tool' IDENT '{' tool_body '}'

tool_body ::= (schema_block | handler_field | security_block)*

schema_block ::= 'schema' '{' schema_field* '}'
schema_field ::= IDENT ':' type_ref

handler_field ::= 'handler' '=' external_handler

external_handler ::= 'external' '(' string_lit ')'

security_block ::= 'security' '{' security_field* '}'

security_field ::= validate_block
                 | sandbox_block
                 | output_block

validate_block ::= 'validate' '{' validate_field* '}'

validate_field ::= IDENT '=' '{' validation_rule* '}'

validation_rule ::= 'max_length' '=' number_lit
                  | 'min_length' '=' number_lit
                  | 'pattern' '=' string_lit
                  | 'deny_patterns' '=' array_lit
                  | 'min' '=' number_lit
                  | 'max' '=' number_lit

sandbox_block ::= 'sandbox' '{' sandbox_field* '}'

sandbox_field ::= 'timeout_ms' '=' number_lit
                | 'max_memory_mb' '=' number_lit
                | 'network_allowed' '=' boolean_lit
                | 'network_hosts' '=' array_lit
                | 'filesystem_allowed' '=' boolean_lit
                | 'filesystem_paths' '=' array_lit
                | 'filesystem_mode' '=' filesystem_mode

filesystem_mode ::= 'readonly' | 'readwrite'

output_block ::= 'output' '{' output_field* '}'

output_field ::= 'max_size_kb' '=' number_lit
               | 'schema' '=' reference
```

### 3.4 Event Block

```ebnf
event_decl ::= 'event' IDENT '{' event_body '}'

event_body ::= 'payload' '=' reference
```

### 3.5 Workflow Block

```ebnf
workflow_decl ::= 'workflow' IDENT '{' workflow_body '}'

workflow_body ::= steps_block
                | steps_block returns_field

steps_block ::= 'steps' '{' step_decl* '}'

step_decl ::= IDENT '=' step_invocation

step_invocation ::= tool_invocation
                  | capability_invocation
                  | agent_invocation

tool_invocation       ::= 'tool'      IDENT invocation_args
capability_invocation ::= 'capability' IDENT invocation_args
agent_invocation      ::= 'agent'     IDENT invocation_args

invocation_args ::= '{' invocation_field* '}'
invocation_field ::= IDENT '=' value

returns_field ::= 'returns' '=' reference
```

### 3.6 Data Block

```ebnf
data_decl ::= 'data' IDENT '{' data_field* '}'

data_field ::= IDENT ':' type_ref
```

### 3.7 Type References

```ebnf
type_ref ::= primitive_type
           | array_type
           | object_type
           | reference       // data.SomeType

primitive_type ::= 'string'
                 | 'number'
                 | 'boolean'

array_type ::= 'array' '<' type_ref '>'

object_type ::= 'object' '{' data_field* '}'
```

### 3.8 Config Block

```ebnf
config_decl ::= 'config' IDENT '{' config_body '}'

config_body ::= (config_field | config_nested_block)*

config_field ::= IDENT '=' value

config_nested_block ::= IDENT '{' config_field* '}'
```

---

### 3.9 Provider Block

```ebnf
provider_decl ::= 'provider' IDENT '{' provider_body '}'

provider_body ::= provider_field*

provider_field ::= 'type' '=' provider_type
                 | 'credentials' '=' (credential_ref | credentials_block)
                 | config_nested_block
                 | limits_block

provider_type ::= 'llm' | 'embedding' | 'vision' | 'audio'

credential_ref ::= env_ref | secrets_ref

env_ref ::= 'env' '.' IDENT

secrets_ref ::= 'secrets' '.' IDENT

credentials_block ::= '{' credential_field* '}'

credential_field ::= IDENT '=' credential_ref

limits_block ::= 'limits' '{' limit_field* '}'

limit_field ::= 'requests_per_minute' '=' number_lit
              | 'tokens_per_minute' '=' number_lit
              | 'requests_per_day' '=' number_lit
```

---

### 3.10 Policy Block

```ebnf
policy_decl ::= 'policy' IDENT '{' policy_body '}'

policy_body ::= (allow_block | deny_block | limits_block)*

allow_block ::= 'allow' '{' allow_field* '}'

allow_field ::= 'tools' '=' array_lit
              | 'workflows' '=' array_lit
              | 'data' '=' array_lit
              | 'capabilities' '=' array_lit

deny_block ::= 'deny' '{' deny_field* '}'

deny_field ::= 'tools' '=' array_lit
             | 'workflows' '=' array_lit
             | 'data' '=' array_lit

limits_block ::= 'limits' '{' limit_field* '}'

limit_field ::= 'max_memory_mb' '=' number_lit
              | 'max_execution_time' '=' number_lit
              | 'max_tool_calls' '=' number_lit
              | 'max_workflow_depth' '=' number_lit
```

---

### 3.11 Template Block

```ebnf
template_decl ::= 'template' IDENT '{' template_body '}'

template_body ::= template_field*

template_field ::= 'system' '=' string_lit
                 | 'user_prefix' '=' string_lit
                 | 'user_suffix' '=' string_lit
                 | 'format' '=' string_lit
```

---

## 4. Comments

```ebnf
comment ::= '//' (any char until newline)
```

---

## 5. Whitespace

Whitespace is ignored except inside quoted strings.

```ebnf
WS ::= (' ' | '\t' | '\n' | '\r')+
```

---

## 6. Grammar Validity Summary
*   Deterministic: yes
*   LALR(1) compatible: yes
*   No left recursion
*   Fully parseable by ANTLR, Tree-sitter, PEG, Recursive Descent
