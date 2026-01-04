# REI Design Principles

*This document translates the REI Manifesto into actionable design principles for implementers (human and AI).*

---

## Principle 1: Everything Composes

**From Manifesto**: *Composition over Abstraction*

**What this means in practice**:

- Every operation should accept input and produce output in compatible forms
- The stack is the universal interface - data flows through it
- No special cases: if it can't work on the stack, redesign it
- Composition should be **mechanical** - no hidden conversions or coercions

**Design tests**:
- Can two words be combined without glue code?
- Can the result be used as input to another word?
- Is the stack state predictable after execution?

**Anti-patterns**:
- Words that only work in specific contexts
- Operations that require "setup" beyond stack state
- Side effects that break composition (use explicit state words instead)

---

## Principle 2: Visibility Over Convenience

**From Manifesto**: *The Stack Is the Interface* + *Execution Must Be Observable*

**What this means in practice**:

- Prefer explicit stack manipulation over implicit variable passing
- Every state change must be observable through the monitor
- No hidden allocations, no surprise mutations
- Performance < Clarity < Inspectability

**Design tests**:
- Can REIMON show what's happening at each step?
- Can execution be paused and stack state examined?
- Are side effects visible before they occur?

**Anti-patterns**:
- Clever optimizations that hide control flow
- Implicit state (globals, closures without stack representation)
- Operations that "just work" but can't be explained

---

## Principle 3: Data and Code Are Peers

**From Manifesto**: *Code Is Data, Data Is Code*

**What this means in practice**:

- Word definitions are data structures that can be inspected
- The dictionary is queryable at runtime
- New words can be synthesized from existing words programmatically
- Execution and introspection use the same mechanisms

**Design tests**:
- Can a word examine its own definition?
- Can the LLM query "what words use X"?
- Can words be modified while the system runs?

**Anti-patterns**:
- Compiled code that's opaque at runtime
- Distinction between "user code" and "system code"
- Irreversible optimizations that lose semantic information

---

## Principle 4: Optimize for Machine Reasoning

**From Manifesto**: *Language Models Are First-Class Developers*

**What this means in practice**:

- Syntax is regular and predictable (no special cases)
- Error messages include full context (stack state, recent words)
- Operations are named for what they do, not abbreviations
- The system explains itself in terms an LLM understands

**Design tests**:
- Can an LLM predict the result without executing?
- Are error messages actionable for correction?
- Can the LLM discover capabilities by examining the dictionary?

**Examples**:

Good:
```
CREATE-ELEMENT           ( tag-name -- element )
APPEND-CHILD            ( parent child -- parent )
```

Problematic:
```
CE                      ( ? -- ? )
>>                      ( mysterious -- stack-effects )
```

**Anti-patterns**:
- Terse names that require human cultural knowledge
- Inconsistent parameter ordering
- Magic behaviors that can't be predicted from signatures

---

## Principle 5: Failures Are Data

**From Manifesto**: *Virtual Machines Are Boundaries of Meaning*

**What this means in practice**:

- Errors don't crash - they become stack values or observable states
- The monitor can inspect failed execution
- Recovery is explicit, not automatic
- The boundary between success and failure is clear

**Design tests**:
- Can the LLM handle the error programmatically?
- Is the error state inspectable through REIMON?
- Can execution continue after handling?

**Error vocabulary**:
```
TRY                     ( word -- result | error-flag )
CATCH                   ( error-flag -- )
ERROR-CONTEXT           ( -- stack-snapshot word-trace )
```

**Anti-patterns**:
- Exceptions that unwind invisibly
- Silent failure modes
- Error handling that requires special syntax

---

## Principle 6: The VM Is the Contract

**From Manifesto**: *Virtual Machines Are Boundaries of Meaning*

**What this means in practice**:

- REIVM defines what execution means (not the host language)
- Stack depth limits are enforced
- Memory is bounded and observable
- The VM can be paused, inspected, resumed

**Invariants**:
- Stack depth never exceeds VM limits
- Dictionary modifications are atomic
- Execution is deterministic given initial state
- The monitor can serialize complete VM state

**Design tests**:
- Can the VM be saved and restored?
- Can execution be replayed from a snapshot?
- Are limits enforced before overflow occurs?

**Anti-patterns**:
- Leaking host language capabilities into VM
- Unbounded recursion
- State that exists outside VM control

---

## Principle 7: Boring Is Better

**From Manifesto**: *Simplicity Is a Survival Strategy*

**What this means in practice**:

- Prefer proven patterns over novelty
- Use straightforward algorithms (even if slower)
- Avoid metaprogramming when direct code works
- Choose clarity over terseness

**Implementation choices**:
- Linear search over hash tables (if dictionary is small)
- Explicit loops over recursion (unless tail-call-optimized)
- Named words over anonymous blocks
- Direct implementation over DSL

**Design tests**:
- Can a new contributor understand this in 5 minutes?
- Will this still make sense in 5 years?
- Can this be explained without diagrams?

**Anti-patterns**:
- "Elegant" solutions that obscure intent
- Premature optimization
- Abstraction for abstraction's sake

---

## Principle 8: Silence Is Intentional

**From Manifesto**: *Silence Is a Feature*

**What this means in practice**:

- No logging unless explicitly requested
- No warnings for valid operations
- Output is data, not commentary
- The monitor observes, doesn't narrate

**Communication contract**:
- Words execute silently (unless they produce output by design)
- Errors are data on the stack, not printed messages
- REIMON shows state when asked, not continuously
- No "helpful" suggestions or tips

**Design tests**:
- Does this word print unnecessarily?
- Can this operation complete without side effects?
- Is this message informational or essential?

**Anti-patterns**:
- Console.log debugging left in production
- "Success!" messages
- Progress indicators for fast operations

---

## Principle 9: Constraints Enable Evolution

**From Manifesto**: *REI Is a Beginning*

**What this means in practice**:

- Start minimal - add only what's proven necessary
- Extension points are explicit (not "plugin architecture")
- Breaking changes are acceptable in early stages
- The LLM learns from failures, not documentation

**Evolution strategy**:
- Implement the smallest working system first
- Let usage patterns reveal missing primitives
- Refactor aggressively based on LLM interaction logs
- Document what *is*, not what might be

**Design tests**:
- Can this be removed without breaking core functionality?
- Is this solving a real problem or an imagined one?
- Can this wait until we need it?

**Anti-patterns**:
- Premature generalization
- "Future-proofing" through complexity
- Features built on speculation

---

## Using These Principles

When implementing any REI component:

1. **Start with the manifesto** - understand the philosophical commitment
2. **Apply these principles** - they constrain your design space productively
3. **Check the specifications** - ensure you meet technical contracts
4. **Reference component specs** - follow established patterns

**For LLMs working on REI**:

These principles are your **decision framework**. When multiple implementations are possible, choose the one that:
- Composes better
- Is more visible
- Treats code as data
- Is more predictable for machines
- Makes failures observable
- Respects the VM boundary
- Is simpler
- Is quieter
- Is more constrained

When in doubt, **do less**.

---

*These principles are living. They will evolve as REI does.*
*But they will always serve the manifesto.*