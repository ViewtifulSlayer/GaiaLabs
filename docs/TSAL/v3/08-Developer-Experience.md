# 8. Developer Experience (DX)

TSAL v3 provides **first-class IDE support** through TypeScript Language Server integration, making assembly development as productive as modern application development.

## TypeScript Language Server Integration

Because TSAL is valid TypeScript, it gets all the benefits of the powerful TypeScript Language Server, which is supported by most modern code editors like VS Code, Sublime Text, and Vim.

### Intelligent Autocomplete

```typescript
// Memory address autocomplete
LDA(gameAddresses.  // Shows: directPage, absolute with detailed documentation

// Instruction autocomplete with parameter hints
L    // Shows: LDA, LDX, LDY, LSR, etc.
ctx.LDA( // Shows: $(address), imm(immediate), gameAddresses.symbol, etc.

// Struct constructor autocomplete  
h_  // Shows: h_actor, h_thinker, etc.
h_actor( // Shows: (id: ImmediateByte, type: ImmediateByte, flags: ImmediateByte)

// COP command autocomplete
iogcop.  // Shows: initializeEntity, setDialogueState, callFunction, etc.
```

### Real-Time Error Checking

```typescript
// Type errors shown immediately
ctx.LDA(imm(0x12345));           // ❌ "Value 0x12345 too large for 16-bit immediate"
ctx.STA(imm(42));                // ❌ "Cannot store to immediate value"  
ctx.BRA(gameAddresses.absolute.player_data); // ❌ "Cannot branch to data address"

// Memory validation
LDA(dc(0x999));              // ❌ "Direct page offset 0x999 outside valid range (0x00-0xFF)"

// String command validation
WideString`[INVALID:123]Hello[END]`; // ❌ "Unknown command 'INVALID' in WideString"
WideString`[TPL:A:B]Hello[END]`;     // ❌ "Too many parameters for TPL command"
```

### Hover Documentation

```typescript
// Hovering over instructions shows documentation
ctx.LDA(gameAddresses.absolute.player_health);
//  ↑ Hover shows: "Load Accumulator
//     Loads value from memory into accumulator
//     Affects: N, Z flags
//     Cycles: 3 (absolute addressing)"

// Hovering over memory locations shows context
gameAddresses.absolute.player_health
//     ↑ Hover shows: "Player Health Points
//        Location: $09A2
//        Size: 2 bytes (word)"

// Hovering over string commands shows documentation
[LU1:BB]  // ↑ Hover: "Lookup table 1 - Retrieves text from lookup table 1 at index BB"
```

### Refactoring Support

Standard TypeScript refactoring tools work out-of-the-box.

- **Rename Symbol**: Renaming a label, address, or struct in one place will update it across the entire project.
- **Extract Function/Method**: Common sequences of assembly can be extracted into a reusable function (a macro) automatically by the IDE.

### Code Navigation

- **Go to Definition**: Click on any symbol (a label, an address, a struct name) to jump directly to its definition.
- **Find All References**: Right-click any symbol to find every place it's used in the codebase. 