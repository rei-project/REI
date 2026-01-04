# REIVM Specification

*The virtual machine. The executor. The semantic boundary.*

**Repository**: `REIVM/`  
**Depends on**: REI design principles  
**Used by**: REIMON, MCP interface, host applications  

---

## Purpose

REIVM is the execution environment that gives meaning to REI words. It:
- Manages stacks (data and return)
- Maintains the dictionary
- Executes words
- Enforces limits and invariants
- Provides hooks for monitoring and introspection

**What it is NOT**:
- An interpreter (it executes, doesn't parse)
- An optimizer (correctness > speed)
- A user interface (that's REIMON)

---

## Architecture

```
┌─────────────────────────────────────┐
│           REIVM                     │
│                                     │
│  ┌──────────┐      ┌──────────┐   │
│  │  Stacks  │      │Dictionary│   │
│  │  - Data  │      │  - Words │   │
│  │  - Return│      │  - Meta  │   │
│  └──────────┘      └──────────┘   │
│                                     │
│  ┌──────────────────────────────┐  │
│  │    Execution Engine          │  │
│  │    - Fetch  - Execute        │  │
│  │    - Control flow            │  │
│  └──────────────────────────────┘  │
│                                     │
│  ┌──────────────────────────────┐  │
│  │    Observer Interface        │  │
│  │    - Inspect  - Trace        │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
         ↓          ↑
    Execute     Observe
         ↓          ↑
      REIMON / MCP / Host
```

---

## Core Data Structures

### Stack

**Implementation**:
```javascript
class Stack {
  constructor(maxDepth = 256) {
    this.items = [];
    this.maxDepth = maxDepth;
  }

  push(value) {
    if (this.items.length >= this.maxDepth) {
      throw new StackOverflow();
    }
    this.items.push(value);
  }

  pop() {
    if (this.items.length === 0) {
      throw new StackUnderflow();
    }
    return this.items.pop();
  }

  peek() {
    if (this.items.length === 0) {
      throw new StackUnderflow();
    }
    return this.items[this.items.length - 1];
  }

  depth() {
    return this.items.length;
  }

  clear() {
    this.items = [];
  }

  snapshot() {
    return [...this.items];
  }
}
```

**Rationale**:
- Simple array implementation (boring is better)
- Explicit bounds checking (fail fast)
- Snapshot support (for debugging/trace)

### Dictionary

**Implementation**:
```javascript
class Dictionary {
  constructor() {
    this.words = new Map();
    this.definitionOrder = [];
  }

  define(name, word) {
    if (this.words.has(name) && word.protected) {
      throw new Error(`Cannot redefine protected word: ${name}`);
    }
    
    if (this.words.has(name)) {
      word.metadata.previousDefinitions = word.metadata.previousDefinitions || [];
      word.metadata.previousDefinitions.push({
        timestamp: new Date(),
        definition: this.words.get(name)
      });
    }

    this.words.set(name, word);
    if (!this.definitionOrder.includes(name)) {
      this.definitionOrder.push(name);
    }
  }

  find(name) {
    return this.words.get(name) || null;
  }

  forget(name) {
    const word = this.words.get(name);
    if (!word) {
      throw new Error(`Word not found: ${name}`);
    }
    if (word.protected) {
      throw new Error(`Cannot forget protected word: ${name}`);
    }
    this.words.delete(name);
    this.definitionOrder = this.definitionOrder.filter(n => n !== name);
  }

  list(filter = null) {
    if (filter) {
      return this.definitionOrder.filter(name => 
        this.words.get(name).category === filter
      );
    }
    return [...this.definitionOrder];
  }
}
```

**Word structure**:
```javascript
{
  name: string,
  stackEffect: string,        // "( a b -- c )"
  body: Word[] | Function,    // Composite or native
  immediate: boolean,
  protected: boolean,         // Core words
  category: 'CORE' | 'STANDARD' | 'USER',
  metadata: {
    defined: Date,
    usageCount: number,
    previousDefinitions: Array,
    documentation: string     // Optional
  }
}
```

---

## Execution Engine

### Word Execution

```javascript
class VM {
  constructor() {
    this.dataStack = new Stack(256);
    this.returnStack = new Stack(256);
    this.dictionary = new Dictionary();
    this.tracing = false;
    this.trace = [];
  }

  execute(wordOrName) {
    const word = typeof wordOrName === 'string' 
      ? this.dictionary.find(wordOrName)
      : wordOrName;

    if (!word) {
      throw new Error(`Word not found: ${wordOrName}`);
    }

    if (this.tracing) {
      this.trace.push({
        word: word.name,
        stackBefore: this.dataStack.snapshot(),
        timestamp: Date.now()
      });
    }

    // Execute native or composite
    if (typeof word.body === 'function') {
      word.body.call(this); // Native word
    } else {
      for (const subWord of word.body) {
        this.execute(subWord);
      }
    }

    word.metadata.usageCount++;

    if (this.tracing) {
      this.trace[this.trace.length - 1].stackAfter = 
        this.dataStack.snapshot();
    }
  }

  // Convenience method for MCP
  run(code) {
    const words = this.parse(code);
    for (const word of words) {
      this.execute(word);
    }
  }

  parse(code) {
    // Simple whitespace tokenizer for now
    return code.trim().split(/\s+/).filter(Boolean);
  }
}
```

### Control Flow Implementation

**IF/THEN**:
```javascript
// IF is immediate word
dictionary.define('IF', {
  name: 'IF',
  stackEffect: '( flag -- )',
  immediate: true,
  body: function() {
    const flag = this.dataStack.pop();
    if (!flag) {
      // Skip to THEN
      this.skipUntil('THEN');
    }
  }
});
```

**BEGIN/UNTIL**:
```javascript
dictionary.define('BEGIN', {
  name: 'BEGIN',
  immediate: true,
  body: function() {
    this.returnStack.push(this.instructionPointer);
  }
});

dictionary.define('UNTIL', {
  name: 'UNTIL',
  stackEffect: '( flag -- )',
  immediate: true,
  body: function() {
    const flag = this.dataStack.pop();
    if (!flag) {
      this.instructionPointer = this.returnStack.peek();
    } else {
      this.returnStack.pop();
    }
  }
});
```

---

## Observer Interface

### Inspection Methods

```javascript
class VM {
  // ... previous methods ...

  inspectStack() {
    return {
      depth: this.dataStack.depth(),
      contents: this.dataStack.snapshot(),
      max: this.dataStack.maxDepth
    };
  }

  inspectDictionary() {
    return {
      wordCount: this.dictionary.words.size,
      words: this.dictionary.list().map(name => ({
        name,
        ...this.dictionary.find(name)
      }))
    };
  }

  inspectWord(name) {
    const word = this.dictionary.find(name);
    if (!word) return null;

    return {
      name: word.name,
      stackEffect: word.stackEffect,
      definition: this.wordToString(word),
      metadata: word.metadata,
      immediate: word.immediate,
      protected: word.protected
    };
  }

  wordToString(word) {
    if (typeof word.body === 'function') {
      return '[NATIVE]';
    }
    return word.body.map(w => w.name || w).join(' ');
  }

  getTrace() {
    return [...this.trace];
  }

  clearTrace() {
    this.trace = [];
  }

  enableTrace() {
    this.tracing = true;
  }

  disableTrace() {
    this.tracing = false;
  }
}
```

---

## Serialization

### State Export

```javascript
class VM {
  serialize() {
    return {
      version: '0.1.0',
      timestamp: new Date().toISOString(),
      stack: this.dataStack.snapshot(),
      dictionary: this.serializeDictionary(),
      config: {
        stackDepth: this.dataStack.maxDepth,
        returnStackDepth: this.returnStack.maxDepth
      }
    };
  }

  serializeDictionary() {
    return this.dictionary.list()
      .filter(name => {
        const word = this.dictionary.find(name);
        return word.category !== 'CORE'; // Don't serialize native words
      })
      .map(name => {
        const word = this.dictionary.find(name);
        return {
          name: word.name,
          stackEffect: word.stackEffect,
          body: this.wordToString(word),
          immediate: word.immediate,
          metadata: word.metadata
        };
      });
  }

  deserialize(state) {
    if (state.version !== '0.1.0') {
      throw new Error('Incompatible state version');
    }

    this.dataStack.clear();
    state.stack.forEach(value => this.dataStack.push(value));

    // Redefine user words
    state.dictionary.forEach(wordData => {
      // Parse body back into word references
      const body = this.parse(wordData.body);
      this.dictionary.define(wordData.name, {
        ...wordData,
        body,
        category: 'USER'
      });
    });
  }
}
```

---

## Error Handling

### Error Types

```javascript
class StackOverflow extends Error {
  constructor() {
    super('Stack overflow');
    this.type = 'STACK_OVERFLOW';
  }
}

class StackUnderflow extends Error {
  constructor() {
    super('Stack underflow');
    this.type = 'STACK_UNDERFLOW';
  }
}

class WordNotFound extends Error {
  constructor(name) {
    super(`Word not found: ${name}`);
    this.type = 'WORD_NOT_FOUND';
    this.word = name;
  }
}

class ControlFlowError extends Error {
  constructor(message) {
    super(message);
    this.type = 'CONTROL_FLOW_ERROR';
  }
}
```

### Error Context

```javascript
class VM {
  executeWithErrorContext(word) {
    try {
      this.execute(word);
      return { success: true };
    } catch (error) {
      return {
        success: false,
        error: {
          type: error.type || 'UNKNOWN',
          message: error.message,
          context: {
            word: word.name || word,
            stack: this.dataStack.snapshot(),
            dictionarySize: this.dictionary.words.size,
            trace: this.getTrace().slice(-5) // Last 5 steps
          }
        }
      };
    }
  }
}
```

---

## Bootstrap Process

### Minimal Core Words

```javascript
function bootstrapVM() {
  const vm = new VM();

  // Stack manipulation
  vm.dictionary.define('DUP', {
    name: 'DUP',
    stackEffect: '( a -- a a )',
    protected: true,
    category: 'CORE',
    body: function() {
      const a = this.dataStack.peek();
      this.dataStack.push(a);
    }
  });

  vm.dictionary.define('DROP', {
    name: 'DROP',
    stackEffect: '( a -- )',
    protected: true,
    category: 'CORE',
    body: function() {
      this.dataStack.pop();
    }
  });

  vm.dictionary.define('SWAP', {
    name: 'SWAP',
    stackEffect: '( a b -- b a )',
    protected: true,
    category: 'CORE',
    body: function() {
      const b = this.dataStack.pop();
      const a = this.dataStack.pop();
      this.dataStack.push(b);
      this.dataStack.push(a);
    }
  });

  // Arithmetic
  vm.dictionary.define('+', {
    name: '+',
    stackEffect: '( a b -- sum )',
    protected: true,
    category: 'CORE',
    body: function() {
      const b = this.dataStack.pop();
      const a = this.dataStack.pop();
      this.dataStack.push(a + b);
    }
  });

  // ... more CORE words ...

  return vm;
}
```

---

## Implementation Guidelines

### For LLMs implementing REIVM:

1. **Start with stack and dictionary** - these are the foundation
2. **Implement CORE words minimally** - just what's needed
3. **Add execution engine** - simple loop, no optimization
4. **Add observer hooks** - inspection before optimization
5. **Test each word in isolation** - stack effects must be exact
6. **Bootstrap incrementally** - define words in REI using earlier words

### Testing strategy:

```javascript
// Example test
test('DUP duplicates top of stack', () => {
  const vm = bootstrapVM();
  vm.dataStack.push(42);
  vm.execute('DUP');
  assert.equal(vm.dataStack.depth(), 2);
  assert.equal(vm.dataStack.pop(), 42);
  assert.equal(vm.dataStack.pop(), 42);
});
```

### Common pitfalls to avoid:

- **Don't optimize prematurely** - correctness first
- **Don't leak host semantics** - JavaScript `===` is not REI equality
- **Don't hide state** - everything observable
- **Don't make words do too much** - compose, don't conflate

---

## Extension Points

REIVM can be extended with:
- Type checking (optional)
- Async word support
- Compilation to JS
- Multiple VMs (isolation)
- Sandboxing (security)

**All extensions must**:
- Preserve core invariants
- Be documentable
- Be toggleable (don't force on users)

---

## Integration with REIMON

REIMON interacts with REIVM via:

```javascript
const vm = bootstrapVM();

// Execute user input
const result = vm.executeWithErrorContext('MY-WORD');

// Inspect state
const stackState = vm.inspectStack();
const dictState = vm.inspectDictionary();

// Control tracing
vm.enableTrace();
vm.run('5 DUP +');
const trace = vm.getTrace();
```

---

## Success Criteria

REIVM is correct when:
- All CORE words work as specified
- Stack operations never corrupt state
- Dictionary lookups are consistent
- Errors are catchable and informative
- State can be serialized and restored
- REIMON can observe all internal state

**Not required initially**:
- Performance (that comes later)
- Optimization (if ever)
- Advanced features (async, types, etc.)

---

*REIVM is the ground truth. Everything else is interface.*