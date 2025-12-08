# A22 Specification

**Orchestrate agentic systems from natural language**

---

## What is A22?

A22 is a natural language specification for agentic systems.

```a22
agent "assistant"
    knows how to chat
    listens to message.received
    speaks message.reply
    looks at message.* history
```

That's it. The runtime handles the rest.

---

## Philosophy

**Pure events.** Everything is an event in time.

**Natural language.** Keywords like `knows how to`, `listens to`, `speaks`.

**Minimal.** Two constructs: `system` and `agent`. Everything else emerges.

**Emergent behavior.** Systems compose through event relationships, not orchestration.

---

## Files

| File | Description |
|------|-------------|
| **specification-v3.0.md** | Complete language specification |
| **grammar-v3.0.md** | EBNF grammar (minimal, natural language) |
| **runtime-model-v3.0.md** | How runtimes interpret A22 programs |

---

## Quick Start

### Write an agent

```a22
system "hello"
    description: "simple greeting system"

agent "greeter"
    knows how to greet
    listens to user.hello
    speaks greeting.reply
```

### Run it

```bash
a22 run hello.a22
```

---

## Examples

### Simple Chat

```a22
agent "assistant"
    knows how to chat
    listens to message.received
    speaks message.reply
```

### Multi-Agent RAG

```a22
agent "assistant"
    listens to message.received
    speaks retrieval.request

agent "retriever"
    listens to retrieval.request
    speaks retrieval.done

agent "assistant"
    listens to retrieval.done
    speaks answer.generated
```

### Temporal Guarantees

```a22
system "support"
    ensures "all messages receive replies"

when message.received
    the system should eventually speak support.complete
```

---

## Key Concepts

### Events

Everything is an event.

Events don't need to be declared — they emerge from references.

```a22
listens to message.received  # message.received is an event
speaks task.completed       # task.completed is an event
```

### Histories

Event histories are implicit.

```a22
looks at message.* history
```

The runtime maintains all `message.*` events in temporal order.

### Interpretations

Agents interpret histories to extract meaning.

```a22
interprets intent from message.* history
interprets sentiment from conversation history
```

### Guarantees

Systems declare temporal properties.

```a22
when message.received
    the system should eventually speak support.complete
```

---

## Design Principles

1. **Clarity over abstraction** — If you have to guess, we failed.
2. **Intent before mechanics** — Describe what, not how.
3. **Minimalism as power** — Two constructs. Everything else emerges.
4. **Predictability without exceptions** — No special cases.
5. **Everything is temporal** — Events flow forward in time.
6. **Composition over control flow** — Systems emerge from relationships.
7. **Human-readable, machine-rigid** — Readable by anyone, reliable for everyone.

---

## Documentation

- [Full Specification](specification-v3.0.md)
- [Grammar](grammar-v3.0.md)
- [Runtime Model](runtime-model-v3.0.md)
- [Website](https://a22.foundation)

---

## Community

- [GitHub Discussions](https://github.com/a22-foundation/spec/discussions)
- [Contributing Guide](CONTRIBUTING.md)
- [Issue Tracker](https://github.com/a22-foundation/spec/issues)

---

## License

The A22 specification is licensed under **Apache 2.0**.

Code examples in the specification are licensed under **Apache 2.0** or **MIT** (your choice).

---

**A22** — Orchestrate agentic systems from natural language
