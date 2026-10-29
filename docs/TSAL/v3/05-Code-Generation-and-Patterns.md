# 5. Code Generation & Patterns

In TSAL, all output is the result of calling functions within the `layout()` generation context. This section describes the different kinds of emitting functions and demonstrates patterns for creating reusable, type-safe code generation helpers.

## Function Types & Emission

All TSAL constructs are function types that return objects. These objects are then passed to the context (`ctx`) to emit code or data when called within the `layout()` generation context.

| Function Type | Declaration | Emission |
|---|---|---|
| **Label** | `const myLabel = label();` | `ctx(myLabel);` -> Emits a named location in code. |
| **Struct** | `const myActor = h_actor(...);` | `ctx(myActor);` -> Emits the raw byte data of the struct. |
| **String** | `const myStr = WideString\`...\`;` | `ctx(myStr);` -> Emits the encoded string data. |
| **Data** | `const myData = byte(0xDE);` | `ctx(myData);` or `ctx(byte(0xDE, 0xAD));` -> Emits raw bytes. |
| **Instruction** | N/A | `ctx.LDA(imm(5));` -> Emits an assembly instruction. |

### Example of Emission
```typescript
import { LayoutFunction } from 'tsal';
// --- Reference Context ---
export const start = label();
export const playerData = h_actor(imm(1), imm(2), imm(3));

// --- Generation Context ---
export const layout: LayoutFunction = (ctx) => {
    ctx(start);        // Emits the "start:" label
    ctx(playerData);   // Emits bytes: 0x01, 0x02, 0x03
    ctx.LDA(imm(5));    // Emits instruction: A9 05
    ctx.RTL();          // Emits instruction: 6B
};
```

---

## Reusable Code Patterns (Macros)

In TSAL, a "macro" is a standard TypeScript function that returns a `SnesLayoutable` object, typically a `code` block. This approach makes abstractions composable: they can be passed to other functions, stored in variables, or included in data structures like a `pointerTable`.

### Simple, Reusable Functions

You can create functions for common, reusable tasks.

```typescript
const waitForVBlank = () => code(ctx => 
  const loop = label();
  ctx.emit([
    loop,
    LDA.abs(platform.cpu.RDNMI), // Read NMI status
    BPL(loop),                   // Wait for VBlank until NMI bit is set
  ]
));

// Usage: The returned code block is composed into another.
const someGraphicsRoutine = code(ctx => ctx.emit([
  waitForVBlank(), // The function is called, and its result is included
  // ... rest of the code ...
]));
```

### Parameterized Functions

Functions can accept parameters to make them more flexible. These parameters can be TSAL types and are fully type-checked by TypeScript.

```typescript
// A type-safe function for a common arithmetic operation.
// It returns a SnesLayoutable code block.
const plusEqual = (address: Address, value: number) => {
  return code(ctx => ctx.emit([
    LDA.abs(address), // Assuming absolute for simplicity
    CLC(),
    ADC.imm(byte(value)),
    STA.abs(address)
  ]));
};

// Usage is clean and declarative.
const layout = code(ctx => ctx.emit([
  plusEqual(mem.abs.player_health, 10),
  plusEqual(mem.abs.player_score, 100),
]));
```

### Conditional Generation

Since macros are just TypeScript functions, you can use standard `if`/`else` logic to conditionally return different code based on parameters. This is extremely powerful for optimization.

```typescript
// This function returns more optimal code for a factor of 2.
const multiplyBy = (address: Address, factor: number) => {
  if (factor === 2) {
    // If factor is 2, a simple arithmetic shift left is faster.
    return code(ctx => ctx.emit([
      LDA.abs(address),
      ASL(),
      STA.abs(address),
    ]));
  } else {
    // For other factors, return code that calls a general purpose routine.
    return code(ctx => ctx.emit([
      LDA.abs(address),
      LDX.imm(byte(factor)),
      JSR(multiply_routine), // Assumes 'multiply_routine' is defined elsewhere
      STA.abs(address),
    ]));
  }
};

// Usage
const layout = code(ctx => ctx.emit([
  multiplyBy(mem.abs.player_score, 2),  // Generates ASL
  multiplyBy(mem.abs.player_mana, 10),   // Generates JSR
]));
``` 