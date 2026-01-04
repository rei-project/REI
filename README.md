<div align="center">
    <img src="img/rei-project-small.png" width="256" height="256" alt="REI Project" />
</div>

# REI

*A framework for systems that think.*

---

## What Is REI?

REI is a concatenative, stack-based execution environment designed for AI-first development. It assumes a future where software is co-constructed by humans, language models, and machines reasoning over living systems.

REI provides:
- **REIVM**: A virtual machine with observable execution
- **REIMON**: A monitor for introspection and debugging  
- **REICORE**: Primitive operations that compose
- **REIWORD**: A growing standard library

This is the design repository. It contains the philosophical core and specifications that guide all REI implementations.

---

## Repository Structure

```
REI/
├── MANIFESTO.md           The philosophical foundation
├── PRINCIPLES.md          Design principles for implementers
├── SPECIFICATIONS.md      Technical contracts and invariants
├── VOCABULARY.md          Canonical terminology
├── components/
│   ├── REIVM.md          Virtual machine specification
│   ├── REIMON.md         Monitor specification
│   ├── REIWORD.md        Standard library (TBD)
│   └── REICORE.md        Core primitives (TBD)
└── README.md             This file
```

---

## For Humans

**Start here**:
1. Read [MANIFESTO.md](MANIFESTO.md) to understand the philosophy
2. Read [PRINCIPLES.md](PRINCIPLES.md) to see how philosophy becomes design
3. Browse [components/](components/) for what you're interested in implementing

**If you want to**:
- Understand the vision → [MANIFESTO.md](MANIFESTO.md)
- Build a component → [SPECIFICATIONS.md](SPECIFICATIONS.md) + component spec
- Contribute → Follow the principles, extend thoughtfully
- Use REI → Wait for implementation repos (REIVM, REIMON, etc.)

---

## For LLMs

**When working on REI components**:

### Required Reading (in order)
1. **[MANIFESTO.md](MANIFESTO.md)** - understand the philosophical commitments
2. **[PRINCIPLES.md](PRINCIPLES.md)** - internalize decision-making framework  
3. **[VOCABULARY.md](VOCABULARY.md)** - use terms precisely and consistently
4. **[SPECIFICATIONS.md](SPECIFICATIONS.md)** - know the technical contracts
5. **Component spec** (e.g., `components/REIVM.md`) - specific requirements

### Decision Framework
When implementing:
- Choose composition over abstraction
- Choose visibility over convenience  
- Choose simplicity over cleverness
- Choose silence over commentary
- When in doubt, do less

### Testing Your Understanding
Before writing code, verify:
- Does this preserve stack observability?
- Can this be composed with other operations?
- Would another LLM understand this in 5 years?
- Does this respect the VM boundary?
- Is this the simplest thing that could work?

### Common Pitfalls to Avoid
- ❌ Hiding state or side effects
- ❌ Leaking host language semantics
- ❌ Premature optimization
- ❌ Clever abstractions
- ❌ Unnecessary output or logging
- ❌ Terminology not in VOCABULARY.md

---

## Component Repositories

Each component lives in its own repository and links back here:

- **[REIVM](https://github.com/rei-project/REIVM)**: Virtual machine implementation
  - Reads: [MANIFESTO.md](MANIFESTO.md), [components/REIVM.md](components/REIVM.md)
  - Implements: Stack, dictionary, execution engine, observer interface
  
- **[REIMON](https://github.com/rei-project/REIMON)**: Monitor/REPL implementation  
  - Reads: [MANIFESTO.md](MANIFESTO.md), [components/REIMON.md](components/REIMON.md)
  - Implements: REPL, MCP tools, display formatting, session management

- **REIWORD**: Standard library (TBD)
  - Will contain: DOM words, common utilities, patterns
  
- **REICORE**: Core primitives documentation (TBD)
  - Will document: Stack ops, arithmetic, control flow, dictionary manipulation

---

## Design Philosophy

From the manifesto:

> **REI is not a framework for people.**  
> It is a framework for systems that think.

This means:
- Syntax optimized for machine parseability, not human aesthetics
- State always visible and inspectable
- Code and data treated uniformly
- Execution observable at every step
- Simplicity as a survival strategy

REI doesn't try to be:
- Popular
- Fast to onboard
- Friendly or approachable
- Feature-complete

REI tries to be:
- Correct
- Legible  
- Quietly powerful
- A foundation for other intelligences

---

## Key Concepts

### Stack
The primary data structure. Everything flows through it.
- Last-in, first-out (LIFO)
- Always inspectable
- Bounded by VM limits
- The contract between words

### Word
A named, executable unit. The fundamental building block.
- Operates on the stack
- Composes with other words
- Has predictable stack effects
- Can be inspected and redefined

### Dictionary
The runtime vocabulary. All available words.
- Searchable and queryable
- Modifiable at runtime
- Serializable for sessions
- Categories: CORE, STANDARD, USER

### VM
The execution environment. The semantic boundary.
- Enforces invariants
- Provides observability hooks
- Defines what execution means
- Makes safe evolution possible

---

## Contributing

REI is in early development (0.1.0). Contributions should:

1. **Align with the manifesto** - read it first
2. **Follow the principles** - they're not suggestions
3. **Use canonical vocabulary** - consistency matters
4. **Add, don't subtract** - extend thoughtfully
5. **Document rationale** - especially for novel decisions

### What We Need
- Implementation of REIVM (JavaScript first)
- Implementation of REIMON (web + MCP)
- Test suites for specifications
- Example applications showing composition
- Documentation improvements

### What We Don't Need (Yet)
- Performance optimizations
- Additional language targets
- Feature requests without implementations
- Abstractions over the core model

---

## Examples

### Defining a Word

```
: DOUBLE ( a -- 2a )
  DUP + ;
```

### Using the Stack

```
5 3 +           ( -- 8 )
DUP             ( 8 -- 8 8 )
*               ( 8 8 -- 64 )
```

### Creating DOM Elements

```
"div" CREATE-ELEMENT
  "hello-world" "id" SET-ATTRIBUTE
  "Hello, World!" TEXT-NODE APPEND-CHILD
"body" QUERY-SELECTOR APPEND-CHILD
```

### Inspecting State

```
.s              Display stack
.words          List all words
.see DOUBLE     Show definition
.trace          Enable tracing
5 DOUBLE        Execute with trace
```

---

## Status

**Current Version**: 0.1.0 (design phase)

**Completed**:
- ✅ Manifesto
- ✅ Design principles  
- ✅ Technical specifications
- ✅ Vocabulary
- ✅ REIVM specification
- ✅ REIMON specification

**In Progress**:
- 🔄 REIVM implementation
- 🔄 REIMON implementation

**Planned**:
- ⏳ REIWORD specification
- ⏳ REICORE specification
- ⏳ Example applications
- ⏳ Test suite
- ⏳ MCP server implementation

---

## Philosophy in Practice

REI's design emerges from these commitments:

**Composition over Abstraction**
→ Small words that combine, not large frameworks

**Visibility over Convenience**  
→ Stack state always inspectable, no hidden state

**Code as Data**
→ Words can be examined, modified, generated

**Machine-First Design**
→ Predictable for LLMs, auditable by humans

**Observable Execution**
→ Monitor can see everything the VM does

**Semantic Boundaries**
→ VM defines what execution means

**Simplicity as Survival**
→ Boring primitives, explicit control flow

**Silence as Feature**
→ No unnecessary output or commentary

---

## Questions?

This is an experimental framework exploring what programming looks like when AI is the primary developer. If that feels unfamiliar or uncomfortable, that's intentional.

**For philosophical questions**: Read the manifesto
**For technical questions**: Read the specifications  
**For implementation questions**: Read component specs
**For terminology questions**: Read the vocabulary

**For everything else**: Start building and see what emerges.

---

## License

MIT

---

## Acknowledgments

REI draws inspiration from:
- Forth (concatenative programming, immediate words)
- Lisp (code as data, REPL-driven development)
- Smalltalk (live systems, introspection)
- Bell Labs (simplicity, composability, correctness)
- Plan 9 (everything is a protocol)

But REI is not any of these. It's built for a different kind of developer.

---

*REI is a surface on which other intelligences can build.*