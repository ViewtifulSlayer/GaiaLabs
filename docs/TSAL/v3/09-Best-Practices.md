# 9. Best Practices

This section provides recommendations for writing clean, maintainable, and robust TSAL code.

### 1. Use Meaningful Names and Descriptions
Provide clear, descriptive names for modules and detailed descriptions in the `moduleInfo`. This makes the codebase much easier for others (and your future self) to understand.

```typescript
// Good
export const moduleInfo = {
  name: 'PlayerMovement',
  bank: 'C0',
  dependencies: ['InputSystem'],
  description: 'Handles player movement, physics, and collision detection.'
};

// Bad
export const moduleInfo = {
  name: 'mod1',
  description: 'does stuff'
};
```

### 2. Organize Imports Logically
Group imports to make dependencies clear. A good convention is Platform > Game > Local.

```typescript
// Good
import { LDA, STA, RTL, label, imm } from 'c65816';           // Platform imports
import { gameAddresses } from 'iog_memory';                   // Game-specific imports
import { GraphicsMacros } from './graphics-lib';              // Local library imports
```

### 3. Prefer Type-Safe Addressing
Whenever possible, use the symbolic, type-safe addresses provided by `gameAddresses` and `platformAddresses`. This prevents bugs and makes code more readable.

```typescript
// Good - Type-safe, self-documenting, and uses the optimal addressing mode.
const myLayout: LayoutFunction = (ctx) => {
    ctx.LDA(gameAddresses.directPage.actor_flags);
};

// Less Good - Still works, but is less readable and loses type-safety benefits.
const myOtherLayout: LayoutFunction = (ctx) => {
    ctx.LDA(dc(0x10));
};

// Bad - Avoid magic numbers. Prone to errors and hard to maintain.
const myBadLayout: LayoutFunction = (ctx) => {
    ctx.LDA(0x10); // Is this direct page or an immediate value? It's ambiguous.
};
```

### 4. Leverage Conditional Generation
Use standard TypeScript `if/else` statements with a build configuration file to manage platform variations, different ROM versions, or feature flags. This is cleaner and more powerful than preprocessor directives in traditional assemblers.

```typescript
// in build-config.ts
export const REGION = 'US';

// in module.ts
import { REGION } from './build-config';

const getIntroDialogue = () => {
    if (REGION === 'JP') {
        return jp_intro_dialogue;
    } else {
        return us_intro_dialogue;
    }
}
```

### 5. Create Functions for Common Patterns
If you find yourself writing the same sequence of instructions repeatedly, extract it into a function. This is the core principle of macros in TSAL.

```typescript
// Good - A reusable function.
const setBit = (ctx: ExecutionContext, address: Address, bit: number) => {
  ctx.LDA(address);
  ctx.ORA(imm(1 << bit));
  ctx.STA(address);
};

// Usage
const layout: LayoutFunction = (ctx) => {
    setBit(ctx, gameAddresses.directPage.some_flags, 4);
};
```

### 6. Keep `layout()` Functions Focused
The `layout()` function is the entry point for a module. It should be clear and readable, acting as a high-level summary of the module's purpose. Delegate complex logic to helper functions.

```typescript
// Good - High-level and easy to read.
export const layout: LayoutFunction = (ctx) => {
  // Group related operations
  initializePlayer(ctx);
  setupGraphics(ctx);
  loadLevelData(ctx);
  
  // Main game loop
  gameLoop(ctx);
};

// Less Good - All implementation details are inlined.
export const layout: LayoutFunction = (ctx) => {
    ctx.LDA(imm(0));
    ctx.STA(player_x);
    ctx.STA(player_y);
    // ... hundreds more lines of inlined code
}
```

### 7. Naming Conventions

#### Context Object (`ctx`)

When defining a `LayoutFunction`, the execution context parameter should be named `ctx`.

```typescript
// Good
import { LayoutFunction, label } from 'tsal';

const mySubroutine = label('my_subroutine');

const myCode: LayoutFunction = (ctx) => {
  // Emit declarations by passing them to the context
  ctx(mySubroutine);
  
  // Emit instructions by calling methods on the context
  ctx.JSR(mySubroutine);
};

// Avoid - Too platform-specific
const myCodeWithSnes: LayoutFunction = (snes) => { 
  snes.JSR(mySubroutine);
};

// Avoid - Cryptic and less readable
const myCodeWithDollar: LayoutFunction = ($) => {
  $.JSR(mySubroutine);
};
```

**Reasoning:**

*   **Clarity**: `ctx` is a widely recognized abbreviation for "context," making the code's intent clear.
*   **Consistency**: All code and data emission flows through the `ctx` object, creating a single, predictable pattern.
*   **Platform Agnosticism**: It does not tie the code to a specific target like `snes`, aligning with TSAL's core design goal of being system-agnostic. 