# REI Technical Specifications

*Formal contracts and invariants for REI implementations.*

---

## Document Purpose

This specification defines:
- **What must be true** (invariants)
- **What must work** (interfaces)
- **What can be measured** (observability requirements)

Implementations that satisfy these specifications are valid REI systems.

---

## 1. Stack Specifications

### 1.1 Data Stack

**Required operations**:
```
PUSH        ( -- value )          Place value on top of stack
POP         ( value -- )          Remove top value
PEEK        ( -- value )          View top without removing
DEPTH       ( -- n )              Number of values on stack
CLEAR       ( ... -- )            Remove all values
```

**Invariants**:
- Stack has maximum depth D (implementation-defined, minimum 256)
- PUSH on full stack produces overflow error
- POP on empty stack produces underflow error
- DEPTH always returns accurate count
- Stack operations are atomic

**Observability**:
- REIMON can view entire stack contents
- Stack state can be serialized to JSON
- Each value's type is identifiable

### 1.2 Return Stack

**Purpose**: Support word calls and control flow

**Operations**:
```
>R          ( value -- ) [R: -- value]     Move to return stack
R>          ( -- value ) [R: value -- ]    Move from return stack
R@          ( -- value ) [R: value -- value]  Copy from return stack
```

**Invariants**:
- Separate from data stack
- Maximum depth DR (minimum 256)
- Cannot be directly inspected by user words (only by REIMON)
- Used for word return addresses and DO/LOOP counters

### 1.3 Stack Effect Verification

**Contract**:
- Every word must declare its stack effect
- Effect notation: `( inputs -- outputs )`
- Optional side-effect markers: `[prints]`, `[mutates-dom]`, `[async]`

**Enforcement** (optional but recommended):
- REIMON can compare declared vs. actual effects
- Violations can be flagged during execution
- Type checking is optional (but effects must match in count)

---

## 2. Dictionary Specifications

### 2.1 Dictionary Structure

**Required operations**:
```
DEFINE      ( name body -- )      Add word to dictionary
FIND        ( name -- word|null ) Lookup word by name
FORGET      ( name -- )           Remove word from dictionary
WORDS       ( -- names )          List all word names
REDEFINE    ( name body -- )      Replace existing word
```

**Properties**:
- Names are case-sensitive
- Names are unique (REDEFINE handles duplicates)
- Words are stored in definition order
- Dictionary can be serialized

**Invariants**:
- FIND always returns consistent results for same name
- FORGET of nonexistent word is an error
- REDEFINE logs previous definition (for undo/audit)

### 2.2 Word Metadata

**Required fields**:
```javascript
{
  name: string,              // Unique identifier
  stackEffect: string,       // e.g., "( a b -- c )"
  body: [words] | function,  // Definition or native code
  immediate: boolean,        // Executes during definition
  metadata: {
    defined: timestamp,
    redefined: timestamp[],
    usage_count: number      // Optional but useful
  }
}
```

**Access**:
- REIMON can display any word's metadata
- MCP can query metadata programmatically
- LLM can discover word capabilities via metadata

### 2.3 Word Categories

**CORE words**: Implemented in host language
- Cannot be redefined (protected)
- Examples: +, -, DUP, DROP, IF, THEN

**STANDARD words**: Defined in REI itself
- Can be redefined
- Part of REIWORD library
- Examples: TUCK, NIP, ROT

**USER words**: Defined in session
- Full redefine capability
- Not persisted unless session saved
- Examples: application-specific words

---

## 3. Execution Specifications

### 3.1 Execution Model

**Interpretation mode** (REIMON default):
```
1. Read next word name
2. FIND word in dictionary
3. Execute word immediately
4. Repeat
```

**Definition mode** (inside `:` ... `;`):
```
1. Read next word name
2. If immediate word: execute now
3. If normal word: add to current definition
4. Until `;` encountered, then add definition to dictionary
```

**Invariants**:
- Execution is deterministic (same state + input = same output)
- Each word executes atomically
- No concurrent execution within single VM
- Stack state after word execution matches declared effect

### 3.2 Control Flow

**Required structures**:
```
IF ... THEN              ( flag -- )
IF ... ELSE ... THEN     ( flag -- )
BEGIN ... UNTIL          ( -- ) [...( -- flag )]
BEGIN ... AGAIN          ( -- ) [infinite loop]
DO ... LOOP              ( limit start -- )
```

**Invariants**:
- Conditional branches must balance (matched IF/THEN)
- Loops must be properly nested
- Control flow cannot jump between word definitions
- Return stack depth restored after control structures

### 3.3 Error Handling

**Error categories**:
```
STACK_OVERFLOW      Stack depth exceeds maximum
STACK_UNDERFLOW     Pop from empty stack
WORD_NOT_FOUND      FIND returns null
TYPE_ERROR          Operation on incompatible value
CONTROL_FLOW_ERROR  Unmatched IF/THEN, etc.
```

**Error behavior**:
- Errors halt execution immediately
- Stack state preserved at error point
- Error details available to REIMON
- Execution can be resumed or reset

**Error representation** (stack-based):
```
TRY         ( word -- result success-flag )
            If word executes: ( result 1 )
            If error occurs: ( error-info 0 )
```

---

## 4. Monitor Specifications (REIMON)

### 4.1 Required Commands

```
.S          Display stack contents
WORDS       List dictionary words
SEE         ( name -- ) Show word definition
TRACE       Enable execution tracing
NOTRACE     Disable execution tracing
RESET       Clear stack and restore initial dictionary
SAVE        ( filename -- ) Save session state
LOAD        ( filename -- ) Restore session state
```

### 4.2 Display Format

**Stack display**:
```
<5> 10 20 30 40 50
    ^depth  ^top
```

**Word listing**:
```
CORE: + - * / DUP DROP SWAP
STANDARD: TUCK NIP ROT 2DUP
USER: MY-WORD ANOTHER-WORD
```

**Word definition**:
```
SEE TUCK
: TUCK ( a b -- b a b )
  SWAP OVER ;
  [Defined: 2026-01-04T15:23:01Z]
```

### 4.3 Trace Format

**Execution trace**:
```
[1] → DUP         STACK: <2> 5 3
[2] → +           STACK: <3> 5 3 3
[3] → *           STACK: <2> 5 6
                  STACK: <1> 30
```

**Properties**:
- Each line shows word executed and resulting stack
- Trace depth can be limited
- Trace can be exported (for LLM analysis)

---

## 5. MCP Interface Specifications

### 5.1 Required Tools

```
execute_word
  input: { word: string, args?: any[] }
  output: { stack: any[], success: boolean, error?: string }

define_word
  input: { name: string, body: string, stackEffect: string }
  output: { success: boolean, error?: string }

inspect_stack
  input: {}
  output: { stack: any[], depth: number }

inspect_dictionary
  input: { filter?: string }
  output: { words: WordMetadata[] }

get_word_definition
  input: { name: string }
  output: { definition: string, metadata: WordMetadata }

trace_execution
  input: { word: string, args?: any[], trace: boolean }
  output: { result: any, trace: TraceStep[], success: boolean }
```

### 5.2 Response Format

**Success**:
```json
{
  "success": true,
  "data": { ... }
}
```

**Error**:
```json
{
  "success": false,
  "error": {
    "type": "STACK_UNDERFLOW",
    "message": "Attempted to POP from empty stack",
    "context": {
      "word": "MY-WORD",
      "stack": [],
      "dictionary_size": 45
    }
  }
}
```

### 5.3 Serialization Format

**VM State**:
```json
{
  "version": "0.1.0",
  "timestamp": "2026-01-04T15:30:00Z",
  "stack": [5, 10, "hello"],
  "dictionary": [
    {
      "name": "MY-WORD",
      "stackEffect": "( a -- a a )",
      "body": ["DUP"],
      "immediate": false,
      "metadata": { ... }
    }
  ],
  "config": {
    "stack_depth": 256,
    "return_stack_depth": 256
  }
}
```

---

## 6. DOM Integration Specifications (REIWORD)

### 6.1 Required DOM Words

```
CREATE-ELEMENT      ( tag-name -- element )
APPEND-CHILD        ( parent child -- parent )
SET-ATTRIBUTE       ( element attr-name value -- element )
SET-TEXT            ( element text -- element )
QUERY-SELECTOR      ( selector -- element|null )
REMOVE-CHILD        ( parent child -- parent )
```

### 6.2 Element Representation

**On stack**: Elements are opaque references
- Cannot be inspected as raw data
- Must use DOM words to manipulate
- Garbage collected when unreachable

**Properties**:
```
GET-TAG             ( element -- tag-name )
GET-ATTRIBUTE       ( element attr-name -- value|null )
HAS-CHILD           ( parent child -- flag )
```

### 6.3 Event Handling

```
ON-CLICK            ( element word-name -- element )
REMOVE-LISTENER     ( element event-type -- element )
```

**Behavior**:
- Event handler executes named word
- Event object available via EVENT-DATA word
- Handlers execute in VM context (have access to stack/dictionary)

---

## 7. Performance Specifications

### 7.1 Required Performance

These are minimum requirements, not goals:

- Dictionary lookup: O(n) acceptable for n < 1000 words
- Stack operations: O(1)
- Word execution: < 1ms for typical CORE words
- REIMON response: < 100ms for inspection commands

### 7.2 Scaling Limits

**Initial targets** (can be exceeded):
- Dictionary: 10,000 words
- Stack depth: 256 values minimum
- Return stack: 256 values minimum
- Trace buffer: 1,000 steps
- Session state: < 10MB serialized

**Not specified**:
- DOM node limits (browser-dependent)
- Async operation timeouts
- Network request limits

---

## 8. Compliance Testing

An implementation is compliant if:

1. All required operations exist
2. All invariants hold
3. Observability requirements met
4. MCP interface works as specified
5. Error handling follows contract

**Test suite** (to be developed):
- Unit tests for each CORE word
- Integration tests for control flow
- Stack underflow/overflow tests
- Dictionary redefinition tests
- Serialization round-trip tests

---

## 9. Extension Points

Implementations MAY add:

- Additional stack types
- Type checking/inference
- Optimization (compilation to JS)
- Async word support
- Module/namespace system
- Undo/redo for definitions
- Visual debugger integration

**Requirements for extensions**:
- Must not break core specifications
- Must be documentable
- Must be optional (core REI works without them)

---

## 10. Versioning

**Specification version**: 0.1.0 (pre-release)

**Breaking changes** require major version bump.

**Additions** (new required words, etc.) require minor version bump.

**Clarifications** require patch version bump.

---

*This specification is executable - implementations should pass automated tests against it.*