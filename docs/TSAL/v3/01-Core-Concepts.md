# 1. Core Concepts

## Introduction & Philosophy

TypeScript Assembly Language (TSAL) v3 is a modern approach to ROM hacking and assembly code generation. TSAL turns TypeScript into a type-safe assembler, letting you write game code and data as TypeScript modules. The design introduces a clear separation between reference context (declarations) and generation context (emission), making code more maintainable, discoverable, and robust. This approach enables:

- Type-safe assembly and data generation
- IDE integration and autocomplete
- Module-based organization with dependency management
- Easy migration and refactoring

TSAL v3 is designed to be practical for real-world ROM projects, supporting advanced features while keeping the workflow intuitive and safe.

---

## Declarations vs. Definitions (Reference vs. Generation)

TSAL elegantly solves the classic assembly problem of managing complex dependencies and forward references by separating the **declaration** of code and data from their **definition**. This aligns perfectly with the concepts of a Reference Context and a Generation Context.

*   **Reference Context (Declarations)**: This is the "table of contents" for your module. At the top of your file, you use `let` to declare all the code and data blocks that will exist. This allows the rest of the module to *refer* to any block, even before it's defined.
*   **Generation Context (Definitions)**: This is where code and data are actually created. Using the `code()` factory and other helpers, you assign the implementation to the variables you declared earlier. This is where bytes are conceptually *generated*.

This is implemented using a simple and powerful **forward declaration** pattern.

**Example:**
```typescript
// --- Reference Context (Declarations) ---
// Forward-declare all code blocks that will be defined in this file.
export let entryPoint: Code;
export let helperFunction: Code;
export let someData: SnesLayoutable;

// --- Generation Context (Definitions) ---
// Now, define the blocks. They can reference each other without error.

entryPoint = code(ctx => ctx.emit([
    // This call is valid because helperFunction was declared.
    JSL(helperFunction),
    RTL()
]));

helperFunction = code(ctx => ctx.emit([
    // ... some logic that might reference someData ...
    RTS()
]));

someData = byte(0xDE, 0xAD, 0xBE, 0xEF);
```
This pattern is the practical application of the Reference and Generation contexts and is fundamental to writing clean, maintainable TSAL modules. 