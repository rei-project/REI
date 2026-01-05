# REI Vocabulary

*Canonical terms and their precise meanings within the REI framework.*

---

## Core Concepts

### Word
A named, executable unit of functionality. The fundamental building block of REI programs.

**Properties**:
- Has a unique name in the dictionary
- Accepts zero or more stack values as input
- Produces zero or more stack values as output
- May have side effects (must be documented in signature)

**Example**:
```
DUP                     ( a -- a a )
```

### Dictionary
The runtime collection of all defined words. The vocabulary available to execution.

**Properties**:
- Searchable by name
- Queryable for word definitions
- Modifiable at runtime (words can be redefined)
- Serializable (can be saved/loaded)

**Not**:
- A static compilation artifact
- Hidden or opaque
- Limited to "user" vs "system" divisions

### Stack
The primary data structure for passing values between words. Where execution state lives.

**Properties**:
- Last-in, first-out (LIFO)
- Bounded by VM limits
- Contains only values (no types, though values may carry type information)
- Always inspectable

**Notation**:
- `( before -- after )` describes stack effects
- Top of stack is rightmost
- Multiple stacks may exist (data stack, return stack) but are kept separate

### VM (Virtual Machine)
The execution environment that gives meaning to words. The semantic boundary.

**Responsibilities**:
- Maintains the stack(s)
- Holds the dictionary
- Controls execution flow
- Enforces limits and invariants
- Provides the interface for the monitor

**Not responsible for**:
- Optimization (that can come later)
- Pretty printing
- User interaction (that's REIMON's job)

---

## Component Terms

### REIVM
The virtual machine implementation. The executor.

**What it provides**:
- Stack management primitives
- Word execution engine
- Dictionary manipulation
- State introspection hooks

### REIMON
The monitor. The observer and debugger.

**What it provides**:
- REPL interface
- Stack inspection
- Word definition viewing
- Execution tracing
- Breakpoint support (if implemented)

**Not**:
- An IDE
- A code editor
- A tutorial system

### REICORE
The primitive words that cannot be defined in terms of other words. The foundation.

**Examples**:
- Stack manipulation: DUP, DROP, SWAP, OVER, ROT
- Arithmetic: +, -, *, /
- Control flow: IF, THEN, BEGIN, UNTIL
- Dictionary: DEFINE, FORGET

**Characteristics**:
- Implemented in host language (JavaScript initially)
- Small, focused set
- Can be extended, but rarely should be

### REIWORD
The standard library of higher-level words built from REICORE primitives.

**Examples**:
- DOM manipulation: CREATE-ELEMENT, APPEND-CHILD, SET-ATTRIBUTE
- Common patterns: TUCK, NIP, 2DUP
- Utilities: . (print), WORDS (list dictionary)

**Characteristics**:
- Defined in REI itself (not host language)
- Can be modified or replaced
- Grows organically based on usage

---

## Technical Terms

### Stack Effect
The transformation a word performs on the stack. The signature.

**Notation**: `( input₁ input₂ -- output₁ output₂ )`

**Rules**:
- Inputs are consumed left to right (bottom to top)
- Outputs are produced left to right (bottom to top)
- Side effects noted separately: `( a b -- c ) [prints]`

**Examples**:
```
DUP         ( a -- a a )
+           ( a b -- sum )
DROP        ( a -- )
.           ( a -- ) [prints a]
```

### Immediate Word
A word that executes during compilation, not runtime. Used for control structures.

**Examples**:
- IF, THEN, ELSE (compile conditional jumps)
- BEGIN, UNTIL (compile loops)

**Properties**:
- Marked in the dictionary as immediate
- Rare - most words are not immediate
- Used to extend the language itself

### Compilation vs. Interpretation
- **Interpretation**: Words execute immediately as they're encountered (REIMON mode)
- **Compilation**: Words are assembled into new word definitions (when defining)

**Both modes use the same VM** - this is not about performance, it's about when execution happens.

### Host Language
The language in which REIVM is implemented (JavaScript for web, potentially others).

**Important**:
- REI runs *on* the host, but is not *of* the host
- Host capabilities should not leak into REI semantics
- The VM is a boundary - cross it deliberately

---

## Interaction Terms

### Session
A period of interaction with REIMON. Has persistent dictionary state.

**Properties**:
- Can be saved (dictionary snapshot)
- Can be restored
- Can be shared (for debugging or collaboration)

### MCP Interface
The machine-readable protocol through which an LLM interacts with REI.

**Capabilities**:
- Define new words
- Execute words
- Inspect stack and dictionary
- Modify word definitions
- Query for word definitions and usage

**Not**:
- A chat interface
- A natural language parser
- A code generator (the LLM does that)

### Trace
A record of execution showing word calls and stack states.

**Format** (example):
```
STACK: [ 5 3 ]
EXECUTE: +
STACK: [ 8 ]
EXECUTE: DUP
STACK: [ 8 8 ]
```

**Uses**:
- Debugging
- Understanding execution flow
- Training data for LLM learning
- Verification

---

## Anti-Vocabulary

*Terms we explicitly avoid because they carry wrong implications.*

### ❌ "Function"
We say: **Word**

*Why*: Function implies return values, parameters, scope. Words operate on the stack and have effects.

### ❌ "Variable"
We say: **Stack value** or **named location** (if implementing memory words)

*Why*: Variables imply named scope and assignment. REI prefers explicit stack manipulation.

### ❌ "Method" or "Object"
We say: **Word** operating on **values**

*Why*: REI is not object-oriented. Values may be composite, but behavior is in words, not values.

### ❌ "Script" or "Program"
We say: **Word definition** or **word sequence**

*Why*: These terms suggest a distinction that doesn't exist in REI. Everything is words.

### ❌ "Compiler"
We say: **Word definer** or **definition mode**

*Why*: Compilation in REI is just building new words from existing ones, not optimization.

### ❌ "Runtime" vs. "Compile-time"
We say: **Execution** and **Definition**

*Why*: Both happen in the same system. There's no separate compilation phase.

### ❌ "Library" or "Package"
We say: **Word collection** or **dictionary subset**

*Why*: REI doesn't have an import/export model (yet). Words are just... available in the dictionary.

---

## Usage Guidelines for LLMs

When working on REI components:

1. **Use these terms precisely** - they have specific meanings here
2. **Avoid anti-vocabulary** - it leads to wrong mental models
3. **When introducing new terms**, propose them for inclusion here
4. **In documentation and code comments**, use canonical vocabulary
5. **In stack effect notation**, follow the standard format exactly

**Consistency matters** because:
- Multiple LLMs may work on different components
- Humans need to audit and understand the code
- The manifesto's vision requires precision

---

## Term Evolution

This vocabulary will grow as REI does, but changes will be:
- **Deliberate** (not accidental)
- **Documented** (with rationale)
- **Minimal** (prefer existing terms)

Propose new terms by:
1. Explaining what concept isn't covered
2. Showing why existing terms don't fit
3. Suggesting precise definition
4. Providing usage examples

---

*Words matter. Use them carefully.*