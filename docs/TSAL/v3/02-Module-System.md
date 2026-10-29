# 2. Module System

## The TSAL Module Interface

All TSAL modules must conform to a base interface, which ensures they can be discovered and processed by the toolchain. While this can be thought of as a conceptual `TSALModule` interface, the modern and practical implementation is a file that provides a specific named export: `layout`.

A TypeScript file is considered a TSAL module if it exports a constant named `layout` that is a `SnesLayoutable` object.

```typescript
// A SnesLayoutable object has a single method that returns its binary representation.
interface SnesLayoutable {
  layout(ctx: LayoutContext): Uint8Array;
}

// Any module that exports a 'layout' object conforming to this is a valid TSAL module.
export const layout: SnesLayoutable = code(ctx => ctx.emit([
    // ... code and data ...
]));
```

---

## The `layout` Export: The Module Manifest

The `layout` export is the most critical part of any TSAL module. It serves as the **manifest**, telling the toolchain two crucial things:

1.  **What** pieces from the module should be included in the final ROM.
2.  **In what order** those pieces should be laid out in memory.

This explicit ordering eliminates a whole class of bugs common in traditional assembly where the layout of code and data is implicit and fragile. Module dependencies are handled via standard TypeScript `import`/`export` statements, which the toolchain can analyze.

---

## Full Module Example

This example shows all the concepts in practice: importing dependencies, forward-declaring the module's contents, defining the logic, and finally, exporting the `layout` manifest.

```typescript
// st68_lily.ts

// --- Imports ---
// Import CPU instructions, platform helpers, game-specific objects, and memory definitions.
import { code, Code, SnesLayoutable, pointerTable, widestring } from 'platform';
import { LDA, STA, JSL, RTL } from 'op';
import { COP } from 'game';
import { mem } from 'game/memory';

// --- Reference Context: Declarations ---
// Forward-declare all code and data blocks defined in this file.
export let e_st68_lily: Code,
    code_list_06B4FB: SnesLayoutable,
    code_06B501: Code;
    // ... and so on for the rest of the module's content

// --- Generation Context: Code & Data Definitions ---

// The entry point for the module's logic
e_st68_lily = code(ctx => ctx.emit([
    LDA.abs(mem.abs.lily_state_0AA6),
    STA.dp(0x00),
    COP[0xD9](word(0x0000), code_list_06B4FB),
]));

// A type-safe table of pointers to other code blocks.
code_list_06B4FB = pointerTable<Code>([
    code_06B501,
    // ... other code blocks
]);

code_06B501 = code(ctx => ctx.emit([ /* ... instructions ... */ ]));

// --- Module Layout Definition ---
// The final, critical export that defines the module's content and order.
export const layout = code((ctx) => ctx.emit([
    e_st68_lily,
    code_list_06B4FB,
    code_06B501,
    // ... all other pieces from this module
])); 