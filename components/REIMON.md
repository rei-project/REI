# REIMON Specification

*The monitor. The observer. The REPL.*

**Repository**: `REIMON/`  
**Depends on**: REIVM  
**Used by**: Humans, LLMs (via MCP), host applications  

---

## Purpose

REIMON is the interface to REIVM. It:
- Provides interactive execution (REPL)
- Displays stack and dictionary state
- Enables debugging and introspection
- Exposes MCP tools for LLM interaction
- Records and exports session state

**What it is NOT**:
- A code editor
- A tutorial system
- A graphical IDE
- A chat interface

---

## Architecture

```
┌─────────────────────────────────────┐
│           REIMON                    │
│                                     │
│  ┌──────────────────────────────┐  │
│  │   REPL Interface             │  │
│  │   - Read   - Display         │  │
│  └──────────────────────────────┘  │
│                                     │
│  ┌──────────────────────────────┐  │
│  │   MCP Tools                  │  │
│  │   - execute_word             │  │
│  │   - define_word              │  │
│  │   - inspect_*                │  │
│  └──────────────────────────────┘  │
│                                     │
│  ┌──────────────────────────────┐  │
│  │   Session Management         │  │
│  │   - Save   - Load   - Reset  │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
              ↓ ↑
            REIVM
```

---

## REPL Interface

### Basic Loop

```javascript
class REIMON {
  constructor(vm) {
    this.vm = vm;
    this.history = [];
    this.running = false;
  }

  async start() {
    this.running = true;
    console.log('REI Monitor 0.1.0');
    console.log('Type HELP for commands');
    
    while (this.running) {
      const input = await this.readLine('rei> ');
      this.processInput(input);
    }
  }

  processInput(input) {
    if (!input.trim()) return;

    this.history.push(input);

    // Handle monitor commands
    if (input.startsWith('.')) {
      this.handleCommand(input.slice(1));
      return;
    }

    // Execute as REI code
    try {
      this.vm.run(input);
      this.displayStack();
    } catch (error) {
      this.displayError(error);
    }
  }
}
```

### Monitor Commands

Commands are prefixed with `.` to distinguish from REI words:

```javascript
handleCommand(cmd) {
  const [command, ...args] = cmd.split(/\s+/);

  switch (command.toLowerCase()) {
    case 's':
      this.displayStack();
      break;
    
    case 'words':
      this.displayWords(args[0]); // optional filter
      break;
    
    case 'see':
      this.displayWord(args[0]);
      break;
    
    case 'trace':
      this.vm.enableTrace();
      console.log('Tracing enabled');
      break;
    
    case 'notrace':
      this.vm.disableTrace();
      console.log('Tracing disabled');
      break;
    
    case 'reset':
      this.resetSession();
      break;
    
    case 'save':
      this.saveSession(args[0]);
      break;
    
    case 'load':
      this.loadSession(args[0]);
      break;
    
    case 'help':
      this.displayHelp();
      break;
    
    case 'quit':
      this.running = false;
      break;
    
    default:
      console.log(`Unknown command: ${command}`);
      console.log('Type .help for available commands');
  }
}
```

---

## Display Formatting

### Stack Display

```javascript
displayStack() {
  const state = this.vm.inspectStack();
  
  if (state.depth === 0) {
    console.log('<0>');
    return;
  }

  const items = state.contents.map(v => this.formatValue(v));
  console.log(`<${state.depth}> ${items.join(' ')}`);
}

formatValue(value) {
  if (value === null) return 'null';
  if (value === undefined) return 'undefined';
  if (typeof value === 'string') return `"${value}"`;
  if (typeof value === 'object') return '[object]';
  return String(value);
}
```

**Output examples**:
```
<0>
<1> 42
<3> 10 20 30
<2> "hello" "world"
<1> [object]
```

### Word Listing

```javascript
displayWords(category = null) {
  const words = this.vm.inspectDictionary().words;
  
  const filtered = category
    ? words.filter(w => w.category === category.toUpperCase())
    : words;

  const grouped = this.groupByCategory(filtered);

  for (const [cat, wordList] of Object.entries(grouped)) {
    console.log(`${cat}:`);
    const names = wordList.map(w => w.name);
    this.printColumns(names, 4);
    console.log();
  }
}

groupByCategory(words) {
  return words.reduce((acc, word) => {
    const cat = word.category || 'USER';
    if (!acc[cat]) acc[cat] = [];
    acc[cat].push(word);
    return acc;
  }, {});
}

printColumns(items, cols) {
  const rows = Math.ceil(items.length / cols);
  for (let row = 0; row < rows; row++) {
    const rowItems = [];
    for (let col = 0; col < cols; col++) {
      const idx = row + col * rows;
      if (idx < items.length) {
        rowItems.push(items[idx].padEnd(15));
      }
    }
    console.log('  ' + rowItems.join(''));
  }
}
```

**Output example**:
```
CORE:
  +              -              *              /              
  DUP            DROP           SWAP           OVER           
  IF             THEN           BEGIN          UNTIL          

STANDARD:
  TUCK           NIP            ROT            2DUP           

USER:
  MY-WORD        CALCULATE      PROCESS        
```

### Word Definition Display

```javascript
displayWord(name) {
  const word = this.vm.inspectWord(name);
  
  if (!word) {
    console.log(`Word not found: ${name}`);
    return;
  }

  console.log(`: ${word.name} ${word.stackEffect}`);
  console.log(`  ${word.definition}`);
  console.log(`  ;`);
  console.log();
  console.log(`Category: ${word.category || 'USER'}`);
  console.log(`Defined: ${new Date(word.metadata.defined).toISOString()}`);
  console.log(`Usage: ${word.metadata.usageCount} times`);
  
  if (word.immediate) {
    console.log('Immediate: yes');
  }
}
```

**Output example**:
```
: TUCK ( a b -- b a b )
  SWAP OVER
  ;

Category: STANDARD
Defined: 2026-01-04T15:30:00.000Z
Usage: 42 times
```

### Trace Display

```javascript
displayTrace() {
  const trace = this.vm.getTrace();
  
  if (trace.length === 0) {
    console.log('No trace recorded. Enable with .trace');
    return;
  }

  console.log('Execution trace:');
  trace.forEach((step, idx) => {
    const before = step.stackBefore.map(v => this.formatValue(v)).join(' ');
    const after = step.stackAfter.map(v => this.formatValue(v)).join(' ');
    
    console.log(`[${idx + 1}] → ${step.word.padEnd(15)} STACK: <${step.stackBefore.length}> ${before}`);
    
    if (idx === trace.length - 1) {
      console.log(' '.repeat(18) + ` STACK: <${step.stackAfter.length}> ${after}`);
    }
  });
}
```

**Output example**:
```
Execution trace:
[1] → DUP              STACK: <2> 5 3
[2] → +                STACK: <3> 5 3 3
[3] → *                STACK: <2> 5 6
                       STACK: <1> 30
```

### Error Display

```javascript
displayError(error) {
  console.log(`ERROR: ${error.message}`);
  
  if (error.context) {
    console.log(`  at word: ${error.context.word}`);
    console.log(`  stack: <${error.context.stack.length}> ${
      error.context.stack.map(v => this.formatValue(v)).join(' ')
    }`);
  }
  
  if (this.vm.tracing && error.context?.trace) {
    console.log('  recent execution:');
    error.context.trace.forEach(step => {
      console.log(`    ${step.word}`);
    });
  }
}
```

**Output example**:
```
ERROR: Stack underflow
  at word: MY-WORD
  stack: <0>
  recent execution:
    DUP
    DROP
    DROP
    +
```

---

## MCP Interface

### Tool Definitions

```javascript
const MCP_TOOLS = [
  {
    name: 'execute_word',
    description: 'Execute REI code and return result',
    parameters: {
      code: {
        type: 'string',
        description: 'REI code to execute (space-separated words)'
      }
    }
  },
  {
    name: 'define_word',
    description: 'Define a new word in the dictionary',
    parameters: {
      name: {
        type: 'string',
        description: 'Name of the new word'
      },
      stackEffect: {
        type: 'string',
        description: 'Stack effect notation, e.g., "( a b -- c )"'
      },
      body: {
        type: 'string',
        description: 'Word definition (space-separated words)'
      }
    }
  },
  {
    name: 'inspect_stack',
    description: 'Get current stack state',
    parameters: {}
  },
  {
    name: 'inspect_dictionary',
    description: 'List all words or filter by category',
    parameters: {
      category: {
        type: 'string',
        optional: true,
        enum: ['CORE', 'STANDARD', 'USER']
      }
    }
  },
  {
    name: 'get_word_definition',
    description: 'Get definition and metadata for a specific word',
    parameters: {
      name: {
        type: 'string',
        description: 'Name of the word to inspect'
      }
    }
  },
  {
    name: 'trace_execution',
    description: 'Execute code with tracing enabled and return trace',
    parameters: {
      code: {
        type: 'string',
        description: 'REI code to execute'
      }
    }
  },
  {
    name: 'reset_vm',
    description: 'Clear stack and restore initial dictionary state',
    parameters: {}
  }
];
```

### Tool Implementations

```javascript
class MCPInterface {
  constructor(vm) {
    this.vm = vm;
  }

  execute_word({ code }) {
    const stackBefore = this.vm.inspectStack().contents;
    
    try {
      this.vm.run(code);
      return {
        success: true,
        stack: this.vm.inspectStack().contents,
        stackChange: {
          before: stackBefore,
          after: this.vm.inspectStack().contents
        }
      };
    } catch (error) {
      return {
        success: false,
        error: {
          type: error.type || 'UNKNOWN',
          message: error.message,
          context: error.context
        }
      };
    }
  }

  define_word({ name, stackEffect, body }) {
    try {
      const wordBody = this.vm.parse(body);
      
      this.vm.dictionary.define(name, {
        name,
        stackEffect,
        body: wordBody,
        immediate: false,
        category: 'USER',
        metadata: {
          defined: new Date(),
          usageCount: 0
        }
      });

      return {
        success: true,
        message: `Defined: ${name} ${stackEffect}`
      };
    } catch (error) {
      return {
        success: false,
        error: {
          type: 'DEFINITION_ERROR',
          message: error.message
        }
      };
    }
  }

  inspect_stack() {
    const state = this.vm.inspectStack();
    return {
      success: true,
      depth: state.depth,
      contents: state.contents,
      max: state.max
    };
  }

  inspect_dictionary({ category = null }) {
    const dict = this.vm.inspectDictionary();
    
    let words = dict.words;
    if (category) {
      words = words.filter(w => w.category === category);
    }

    return {
      success: true,
      wordCount: words.length,
      totalWords: dict.wordCount,
      words: words.map(w => ({
        name: w.name,
        stackEffect: w.stackEffect,
        category: w.category,
        immediate: w.immediate,
        usageCount: w.metadata.usageCount
      }))
    };
  }

  get_word_definition({ name }) {
    const word = this.vm.inspectWord(name);
    
    if (!word) {
      return {
        success: false,
        error: {
          type: 'WORD_NOT_FOUND',
          message: `Word not found: ${name}`
        }
      };
    }

    return {
      success: true,
      word: {
        name: word.name,
        stackEffect: word.stackEffect,
        definition: word.definition,
        category: word.category,
        immediate: word.immediate,
        metadata: word.metadata
      }
    };
  }

  trace_execution({ code }) {
    this.vm.enableTrace();
    this.vm.clearTrace();
    
    const result = this.execute_word({ code });
    const trace = this.vm.getTrace();
    
    this.vm.disableTrace();

    return {
      ...result,
      trace: trace.map(step => ({
        word: step.word,
        stackBefore: step.stackBefore,
        stackAfter: step.stackAfter
      }))
    };
  }

  reset_vm() {
    this.vm.dataStack.clear();
    // Don't reset dictionary - preserve defined words
    // (true reset would require re-bootstrap)
    
    return {
      success: true,
      message: 'Stack cleared'
    };
  }
}
```

---

## Session Management

### Save Session

```javascript
saveSession(filename) {
  if (!filename) {
    filename = `rei-session-${Date.now()}.json`;
  }

  const state = this.vm.serialize();
  state.history = this.history;

  try {
    const json = JSON.stringify(state, null, 2);
    // In browser: download as file
    // In Node: write to filesystem
    this.downloadFile(filename, json);
    console.log(`Session saved: ${filename}`);
  } catch (error) {
    console.log(`Error saving session: ${error.message}`);
  }
}
```

### Load Session

```javascript
loadSession(filename) {
  try {
    // In browser: file upload
    // In Node: read from filesystem
    const json = this.loadFile(filename);
    const state = JSON.parse(json);

    this.vm.deserialize(state);
    this.history = state.history || [];

    console.log(`Session loaded: ${filename}`);
    console.log(`Restored ${state.dictionary.length} user words`);
  } catch (error) {
    console.log(`Error loading session: ${error.message}`);
  }
}
```

### Reset Session

```javascript
resetSession() {
  const confirm = this.confirm('Reset session? All user words will be lost.');
  if (!confirm) return;

  this.vm = bootstrapVM(); // Re-bootstrap fresh VM
  this.history = [];
  console.log('Session reset');
}
```

---

## Help System

```javascript
displayHelp() {
  console.log(`
REI Monitor Commands (prefix with .):

  .s              Display stack contents
  .words [cat]    List dictionary words (optional category filter)
  .see <word>     Show word definition
  .trace          Enable execution tracing
  .notrace        Disable execution tracing
  .reset          Clear stack and reset session
  .save [file]    Save session to file
  .load <file>    Load session from file
  .help           Show this help
  .quit           Exit monitor

Without . prefix, input is executed as REI code.

Examples:
  5 3 +           Execute words (adds 5 and 3)
  : DOUBLE DUP + ;  Define new word
  .see DOUBLE     View word definition
  .s              Show resulting stack
  `);
}
```

---

## Integration Examples

### Web Integration

```html
<!DOCTYPE html>
<html>
<head>
  <title>REIMON Web</title>
</head>
<body>
  <div id="terminal"></div>
  <input id="input" type="text" />
  
  <script type="module">
    import { bootstrapVM } from './reivm.js';
    import { REIMON } from './reimon.js';

    const vm = bootstrapVM();
    const monitor = new REIMON(vm);
    
    // Connect to DOM
    const terminal = document.getElementById('terminal');
    const input = document.getElementById('input');

    monitor.output = (text) => {
      const line = document.createElement('div');
      line.textContent = text;
      terminal.appendChild(line);
    };

    input.addEventListener('keypress', (e) => {
      if (e.key === 'Enter') {
        monitor.processInput(input.value);
        input.value = '';
      }
    });
  </script>
</body>
</html>
```

### MCP Server

```javascript
import { bootstrapVM } from './reivm.js';
import { MCPInterface } from './reimon.js';

const vm = bootstrapVM();
const mcp = new MCPInterface(vm);

// Expose tools to MCP protocol
server.setRequestHandler('tools/list', async () => ({
  tools: MCP_TOOLS
}));

server.setRequestHandler('tools/call', async (request) => {
  const { name, arguments: args } = request.params;
  
  if (typeof mcp[name] === 'function') {
    return mcp[name](args);
  }
  
  throw new Error(`Unknown tool: ${name}`);
});
```

---

## Testing REIMON

### Interactive Tests

```javascript
test('REPL executes code correctly', async () => {
  const vm = bootstrapVM();
  const monitor = new REIMON(vm);

  // Simulate user input
  monitor.processInput('5 3 +');
  
  assert.equal(vm.dataStack.pop(), 8);
});

test('.s displays stack', () => {
  const vm = bootstrapVM();
  const monitor = new REIMON(vm);
  
  vm.dataStack.push(1);
  vm.dataStack.push(2);
  
  const output = captureOutput(() => monitor.displayStack());
  assert.equal(output, '<2> 1 2');
});
```

### MCP Tests

```javascript
test('execute_word returns correct result', () => {
  const vm = bootstrapVM();
  const mcp = new MCPInterface(vm);

  const result = mcp.execute_word({ code: '5 DUP +' });
  
  assert.equal(result.success, true);
  assert.deepEqual(result.stack, [10]);
});

test('define_word creates new word', () => {
  const vm = bootstrapVM();
  const mcp = new MCPInterface(vm);

  const result = mcp.define_word({
    name: 'DOUBLE',
    stackEffect: '( a -- 2a )',
    body: 'DUP +'
  });

  assert.equal(result.success, true);
  assert.notEqual(vm.dictionary.find('DOUBLE'), null);
});
```

---

## Implementation Guidelines

### For LLMs implementing REIMON:

1. **Start with basic REPL** - read, execute, display
2. **Add monitor commands** - one at a time
3. **Implement MCP tools** - these are what LLMs will use most
4. **Add formatting** - make output readable
5. **Add session management** - save/load last

### Design priorities:

1. **Clarity > cleverness** - obvious output format
2. **Observability > convenience** - show everything
3. **Predictability > features** - consistent behavior

### Common pitfalls:

- **Don't add "helpful" messages** - silence is a feature
- **Don't interpret user intent** - execute exactly what's given
- **Don't hide errors** - display full context
- **Don't optimize display** - show raw truth

---

## Success Criteria

REIMON is correct when:
- Code executes and shows stack state
- Errors display with full context
- All monitor commands work
- MCP tools return correct JSON
- Sessions can be saved and restored
- Help is accurate

**Not required initially**:
- Syntax highlighting
- Auto-completion
- History search
- Visual debugger

---

*REIMON makes REIVM observable. That is its only job.*